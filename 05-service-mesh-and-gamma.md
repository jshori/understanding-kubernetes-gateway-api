# Understanding Kubernetes Gateway API, Part 5: Service Mesh and GAMMA

This is the final part of the "Understanding Kubernetes Gateway API" series. Every part so far has been about traffic coming in from outside the cluster. This part covers something different: traffic that never leaves the cluster at all, service-to-service traffic, and how Gateway API extends to cover that too.

## A new kind of traffic: north-south vs east-west

Everything in Parts 1 through 4 covered one direction of traffic: a client outside the cluster, sending a request in. This kind of traffic has a name, **north-south traffic**, since it's pictured as coming from "above" the cluster, down into it.

There's another kind of traffic that never involves an outside client at all. Let's say the `engineering` team runs two of their own internal services, an `order-service` and an `inventory-service`. When `order-service` needs to check stock, it calls `inventory-service` directly, entirely inside the cluster. No external client is involved anywhere in that call. This is called **east-west traffic**, service-to-service, side-by-side, rather than top-to-bottom.

A **service mesh** manages east-west traffic, the same way a `Gateway` manages north-south traffic. But it's built and configured completely differently, using its own separate tools, with sidecar proxies (a small helper container attached to each service, handling the actual traffic for it) attached to each service. For a long time, if you wanted to manage east-west traffic, you had to learn a mesh's own configuration system, entirely separate from anything Gateway API covered.

That started to change with GAMMA.

## What is GAMMA, and why does it exist

GAMMA is short for **Gateway API for Mesh Management and Administration**. It's a workstream inside the Gateway API project, dedicated to extending Gateway API into service mesh territory, east-west traffic.

Gateway API was originally built for one job: routing traffic from outside the cluster in, north-south traffic. But as more teams started using it, service mesh users, the ones managing east-west traffic, started asking the same question: could Gateway API's resources work for their traffic too?

The official docs explain what happened next: over time, interest from service mesh users prompted the creation of a dedicated initiative in 2022, to define how Gateway API could also be used for inter-service or east-west traffic within the same cluster.

GAMMA isn't a separate project with its own resources. Its job was to figure out how to reuse the Route resources we already know from Part 3, `HTTPRoute`, `GRPCRoute`, `TLSRoute`, `TCPRoute`, and `UDPRoute`, for this completely different kind of traffic.

## Why Gateway and GatewayClass don't show up here

Here's the first surprise. When you configure mesh traffic, you don't use `Gateway` or `GatewayClass` at all. The official reasoning is simple: this is primarily because there will typically only be one mesh active in the cluster, so the `Gateway` and `GatewayClass` resources are not used when working with a mesh.

Recall from Part 1, `GatewayClass` exists to say "which controller handles this," and `Gateway` exists to say "here's a real entry point into the cluster." Neither question makes sense for east-west traffic. There's usually only one mesh running, so there's nothing to choose between. And the traffic never "enters" the cluster from outside, it's already inside, so there's no entry point to define.

## How mesh routing actually works: a Route attaches directly to a Service

If there's no `Gateway` to attach to, what does an `HTTPRoute` attach to instead, for mesh traffic? The Service itself. Let's compare both side by side to see exactly what changes.

Here's the familiar pattern from Part 1, an `HTTPRoute` attaching to a `Gateway`, for north-south traffic:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: engineering-route
  namespace: engineering
spec:
  parentRefs:
    - name: shared-gw
      namespace: infra
  rules:
    - matches:
        - path:
            value: /api
      backendRefs:
        - name: api-service
          port: 8080
```

Here's what happens with this Route in place. A client outside the cluster sends a request to `shared-gw`. The `Gateway` checks its Listeners, finds one this Route is attached to, and hands the request to this `HTTPRoute`. This Route checks the path, sees it matches `/api`, and forwards the request to `api-service`. The client never talks to `api-service` directly, everything passes through the `Gateway` first.

And here's the mesh version, for the `engineering` team's `order-service` calling `inventory-service` entirely inside the cluster:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: inventory-mesh-route
  namespace: engineering
spec:
  parentRefs:
    - name: inventory-service
      kind: Service
      group: ""
  rules:
    - matches:
        - path:
            value: /stock
      backendRefs:
        - name: inventory-service
          port: 8080
```

Here's what happens with this Route in place. `order-service` calls `inventory-service` directly, using its normal internal address, there's no `Gateway` involved at any point. The mesh's own data plane intercepts that call before it reaches `inventory-service`, notices this `HTTPRoute` is attached to it, and checks the request against it. The path matches `/stock`, so the request is allowed through to `inventory-service`. If `order-service` had called a path that didn't match any rule on this Route, the mesh would reject it instead of letting it through.

The only real difference in the YAML is inside `parentRefs`. In the first example, it names a `Gateway` (`shared-gw`), the default `kind` for `parentRefs` when nothing else is specified. In the second, it explicitly sets `kind: Service` and `group: ""` (the core Kubernetes API group, where Service lives), and names `inventory-service` directly instead.

Everything else, `rules`, `matches`, `backendRefs`, works exactly the same way it did in Parts 1 and 3. Whenever anything inside the mesh calls `inventory-service`, this Route's rules apply to that traffic, the same matching and forwarding logic from earlier parts, just triggered by a Service instead of a Gateway.

## What happens with, and without, a Route attached

There are two states `inventory-service` can be in, and it's worth walking through both.

**Before any Route is attached.** Say no `HTTPRoute` points to `inventory-service` yet. `order-service` calls it, the mesh's data plane sees the call, but since there's no Route telling it what to check, it just lets the request through as normal, no matching, no rules, nothing gets rejected. The Service behaves exactly as if the mesh weren't managing it at all.

**After the Route from before is attached.** Now that `inventory-mesh-route` exists, everything changes. Every single call to `inventory-service`, from `order-service` or anyone else, gets checked against that Route's rules first. A call to `/stock` matches, and goes through. A call to some other path that isn't listed in the Route's rules doesn't match anything, and gets rejected outright, it doesn't fall through to some default behavior, it's blocked.

So attaching a Route isn't just "adding an option", it flips a switch. Once even one Route is attached to a Service, that Route becomes the complete rulebook for that Service. Anything not explicitly allowed is refused.

## A current limitation worth knowing

This part of Gateway API is newer, and it comes with a real, officially documented limitation. Let's say both `order-service` and a second service, `shipping-service`, call `inventory-service`, and each wants different timeout settings for their own calls. Per the official docs, this isn't possible today if they're in the same namespace. The official example describes an identical situation with two other services calling a shared one: they aren't currently able to set different timeouts for their calls to that shared service while sharing a namespace, they would need to be moved into separate namespaces to allow this.

This is an active area of ongoing work, not a permanent design decision, but it's worth knowing about if you're planning to rely on this today.

## What stage is this at right now

Mesh support is real and usable, but it's newer than everything else in this series. Per the official implementations documentation, service mesh routing patterns are currently in the experimental channel. Recall from Part 3 what that means: it's stable enough to build on, but still more likely to change than the Standard-channel features covered in Parts 1 through 4.

## Why this design is worth appreciating

Step back and look at what actually happened here. GAMMA didn't invent new resources for mesh traffic. It reused `HTTPRoute`, the exact same resource type from Part 1, and simply let it attach to a `Service` instead of a `Gateway`. The same rules, the same matching logic, the same YAML shape, now cover two completely different kinds of traffic.

That's the practical payoff of everything this series has covered. Learn the resource model once, `GatewayClass`, `Gateway`, Routes, Listeners, cross-namespace references, and it extends to both north-south and east-west traffic without needing a second, unrelated system to learn.

## You've completed the series

Here's a quick look back at everything covered:

- **Part 1** introduced the roles Gateway API is designed around, and the core resources, `GatewayClass`, `Gateway`, and Routes
- **Part 2** went deeper into `Gateway`, Listeners, TLS modes, and the rules that prevent Listeners from conflicting with each other
- **Part 3** covered the different Route types, `HTTPRoute`, `GRPCRoute`, `TLSRoute`, `TCPRoute`, and `UDPRoute`, and how to pick the right one
- **Part 4** covered `ReferenceGrant` and `BackendTLSPolicy`, the two resources that handle cross-namespace access and backend certificate trust
- **Part 5** (this guide) extended everything learned so far into service mesh, east-west traffic

## Sources

- [The GAMMA Initiative, Gateway API official docs](https://gateway-api.sigs.k8s.io/docs/mesh/gamma/)
- [Mesh Overview, Gateway API official docs](https://gateway-api.sigs.k8s.io/docs/mesh/mesh-overview/)
- [Gateway API v0.8.0: Introducing Service Mesh Support, Kubernetes blog](https://kubernetes.io/blog/2023/08/29/gateway-api-v0-8/)
- [Implementations, Gateway API official docs](https://gateway-api.sigs.k8s.io/docs/implementations/list/)
