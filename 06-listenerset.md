# Understanding Kubernetes Gateway API, Part 6: ListenerSet

This is Part 6 of the "Understanding Kubernetes Gateway API" series. Part 2 mentioned `ListenerSet` briefly, as something worth knowing about, without going deeper. This part explains what it actually is, why it exists, and walks through a full example.

## The problem, in simple words

Recall `shared-gw` from Part 2. Every time a new team wanted their own domain, `engineering.example.com`, then `finance.example.com`, Chihiro had to edit `shared-gw` directly, adding a new Listener each time.

This creates a bottleneck. Chihiro's team becomes the only ones allowed to touch `shared-gw`. Every new team, every certificate renewal, every small change, has to go through them first. As more teams join, this gets slower and more painful for everyone.

## What is a ListenerSet

A `ListenerSet` is a separate object that holds Listeners, the same kind of Listeners we've seen inside a `Gateway`, but living outside it, in their own object. It gets attached to an existing `Gateway`, and once attached, its Listeners behave as if they'd been written directly inside that `Gateway`.

The official docs put it simply: it "decouples network listener configurations, such as ports, hostnames, and TLS termination, from the central Gateway resource," so that "teams can independently define and attach groups of listeners to a central, shared Gateway."

In plain terms: instead of asking Chihiro to add a Listener for you, you create your own `ListenerSet`, in your own namespace, and attach it yourself.

## Why this exists

Two problems, solved at once.

**Self-service.** Teams no longer need to wait on Chihiro for every new domain or certificate. They manage their own `ListenerSet`, in their own namespace, without touching `shared-gw` directly.

**The 64-Listener limit.** Recall from Part 2, a `Gateway` can hold a maximum of 64 Listeners. Since `ListenerSet` Listeners live in separate objects, this limit stops being a single, shared ceiling that every team competes for.

## How it connects to what Chihiro has to do first

A `ListenerSet` can't just show up and attach itself, Chihiro still has to explicitly allow it, once, on `shared-gw`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  gatewayClassName: internal-lb-class
  allowedListeners:
    namespaces:
      from: All
  listeners:
    - name: engineering-https
      protocol: HTTPS
      port: 443
      hostname: "engineering.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: engineering-tls-secret
      allowedRoutes:
        namespaces:
          from: All
    - name: finance-https
      protocol: HTTPS
      port: 443
      hostname: "finance.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: finance-tls-secret
      allowedRoutes:
        namespaces:
          from: All
```

Notice the new field, `allowedListeners.namespaces.from: All`. By default, a `Gateway` doesn't allow any `ListenerSet` to attach at all. This is Chihiro's one-time decision, similar in spirit to `allowedRoutes` from Part 1, but this time controlling who can attach entire Listeners, not just Routes.

## Setup happens before any request ever arrives

It's worth being clear about something here, before walking through what a request actually does. The controller isn't searching through namespaces in real time, checking for a matching `ListenerSet` every time a request comes in.

Instead, the controller constantly watches for `ListenerSet` objects, in the background, all the time. The moment a team creates a new `ListenerSet`, the controller notices it right away, checks whether `shared-gw`'s `allowedListeners` permits it, and if so, immediately merges its Listeners into `shared-gw`'s combined list, well before any client ever sends a request.

By the time a real request arrives, there's no searching to do. The proxy already has a single, ready-to-use list that includes every Listener from `shared-gw` itself and from every attached `ListenerSet`. Matching a request to the right Listener is then just a quick lookup against that already-built list, not a live search across namespaces.

## A worked example: the Marketing team self-serves their own domain

Recall from Part 1, the `marketing` team was part of the original story, but never got its own domain the way `engineering` and `finance` did in Part 2. Let's give them one now, using `ListenerSet` instead of asking Chihiro to edit `shared-gw`.

**Step 1: the Marketing team creates their own ListenerSet, in their own namespace:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: ListenerSet
metadata:
  name: marketing-listeners
  namespace: marketing
spec:
  parentRef:
    name: shared-gw
    kind: Gateway
    group: gateway.networking.k8s.io
  listeners:
    - name: marketing-https
      protocol: HTTPS
      port: 443
      hostname: "marketing.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: marketing-tls-secret
```

Notice `marketing-tls-secret` doesn't need a `ReferenceGrant` from Part 4. Since the `ListenerSet` and the Secret it references both live in the same namespace, `marketing`, this stays entirely within the default, same-namespace case, no cross-namespace permission needed.

**Step 2: the Marketing team creates their Route, attaching to the ListenerSet instead of the Gateway directly:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: marketing-route
  namespace: marketing
spec:
  parentRefs:
    - name: marketing-listeners
      kind: ListenerSet
      group: gateway.networking.k8s.io
  hostnames:
    - "marketing.example.com"
  rules:
    - matches:
        - path:
            value: /api
      backendRefs:
        - name: marketing-api-service
          port: 8080
```

Notice `parentRefs` now says `kind: ListenerSet`, not `Gateway`. Chihiro never had to touch anything for this to work, the Marketing team did all of it themselves, in their own namespace.

## What happens when a request actually arrives

Say a client sends a request to `marketing.example.com/api`. Here's the path it takes:

1. The request reaches `shared-gw`. By this point (recall the previous section), the Gateway's own Listeners (`engineering-https`, `finance-https`) and `marketing-listeners`'s Listener (`marketing-https`) have already been merged into one combined list, this happened when `marketing-listeners` was created, not now.
2. SNI selects the right Listener from that already-built list, the same mechanism from Part 2, this time picking `marketing-https` because the SNI matches its hostname.
3. `marketing-https` decrypts the traffic using `marketing-tls-secret`, and confirms the Host header.
4. `marketing-route`, attached to `marketing-listeners`, matches on hostname and the `/api` path.
5. The request is forwarded to `marketing-api-service`.

Notice this is the exact same flow as Part 2's diagrams, the only difference is where the winning Listener actually lives, inside `shared-gw` itself, or inside a `ListenerSet` attached to it. From the request's point of view, there's no difference at all.

## What happens if two Listeners conflict

Since `shared-gw`'s own Listeners and any attached `ListenerSet`'s Listeners are combined, the same distinctiveness rules from Part 2 still apply, but with one added rule for resolving ties. The official docs are specific about the order:

Listeners on the parent Gateway always take priority over Listeners from a `ListenerSet`. Among competing `ListenerSet`s, the one created earliest wins. If two `ListenerSet`s were created at the exact same time, the one whose name comes first alphabetically wins. The loser is marked `Accepted: false` and `Conflicted: true`, it doesn't get silently dropped, its status will clearly show it lost.

This matters in practice: if the Marketing team ever accidentally picked a hostname that Chihiro's own Listener already used directly on `shared-gw`, Chihiro's Listener would always win, protecting the shared infrastructure from being overridden by a self-service mistake.

## What's been covered across this series

This wraps up the "Understanding Kubernetes Gateway API" series. Here's the full picture:

- **Part 1** introduced the roles Gateway API is designed around, and the core resources, `GatewayClass`, `Gateway`, and Routes
- **Part 2** went deeper into `Gateway`, Listeners, TLS modes, and the rules that prevent Listeners from conflicting with each other
- **Part 3** covered the different Route types, `HTTPRoute`, `GRPCRoute`, `TLSRoute`, `TCPRoute`, and `UDPRoute`, and how to pick the right one
- **Part 4** covered `ReferenceGrant` and `BackendTLSPolicy`, the two resources that handle cross-namespace access and backend certificate trust
- **Part 5** extended everything learned so far into service mesh, east-west traffic
- **Part 6** (this guide) covered `ListenerSet`, letting teams self-serve their own Listeners on a shared Gateway

## Sources

- [ListenerSet, Gateway API official docs](https://gateway-api.sigs.k8s.io/reference/api-types/listenerset/)
- [ListenerSet user guide, Gateway API official docs](https://gateway-api.sigs.k8s.io/guides/user-guides/listener-set/)
- [GEP-1713: ListenerSets, Gateway API official docs](https://gateway-api.sigs.k8s.io/geps/gep-1713/)
- [Gateway API v1.5: Moving features to Stable, Kubernetes blog](https://kubernetes.io/blog/2026/04/21/gateway-api-v1-5/)
- [Gateway API v1.3.0: Gateway Merging, Kubernetes blog](https://kubernetes.io/blog/2025/06/02/gateway-api-v1-3/)
