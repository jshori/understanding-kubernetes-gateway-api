# Understanding Kubernetes Gateway API, Part 2: GatewayClass and Gateway, In Depth

This is Part 2 of the "Understanding Kubernetes Gateway API" series. Part 1 covered who owns what: `GatewayClass`, `Gateway`, and Routes. It used one running example: Chihiro's shared Gateway, `shared-gw`. Engineering, Finance, and HR all used it.

This part goes deeper into `Gateway`. It focuses on **Listeners**. A `Gateway` can have more than one Listener. Kubernetes has strict rules about when that's allowed.

## A quick recap: what's inside a Gateway

Recall `shared-gw` from Part 1. Chihiro created it in the `infra` namespace. Engineering, Finance, and HR each had their own namespace, `engineering`, `finance`, and `hr`. Each team attached their own Route to `shared-gw`, from inside their own namespace.

A `Gateway` has three main parts: `gatewayClassName`, `listeners`, and `addresses`.

Let's go through each one.

**1. `gatewayClassName`**

It says which template to use.

Think back to Part 1. `GatewayClass` was a showroom sign. It said "we can build Sedans." `Gateway` was you ordering one.

The `gatewayClassName` field is the direct link between the two. It's how the `Gateway` says "build me using that specific template."

For example: Ian's team made a `GatewayClass`. They called it `internal-lb-class`. Chihiro's `Gateway` must name it. Exactly like this:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: internal-lb-class
spec:
  controllerName: example.com/gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  gatewayClassName: internal-lb-class   # this is the link to the GatewayClass above
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

(One small note on the example above: `listeners` is a required field, a `Gateway` can't exist without at least one. We've included a minimal one here so the YAML is valid, even though this example is really about `gatewayClassName`. `listeners` gets its own full explanation next.)

Here's why this field matters so much. A controller only pays attention to `Gateway` objects that point to a `GatewayClass` it recognizes as its own. It ignores everything else, it doesn't even look at them.

So say Chihiro left `gatewayClassName` empty, or misspelled it. No controller would recognize this `Gateway` as its own. No controller would pick it up. Nothing would happen. The `Gateway` object would just sit there, doing nothing, no matter how correctly the rest of it was written. This is why `gatewayClassName` can't be left out, without it, there's no controller listening for this `Gateway` at all.

**2. `listeners`**

A list of rules. Each rule describes what traffic the Gateway will accept.

Here's the simplest version, just one rule, accepting plain HTTP on port 80:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  gatewayClassName: internal-lb-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

In Part 1, `shared-gw` only had one Listener like this, a single HTTPS entry on port 443. All three teams shared it.

But say the `engineering` team wants their own domain now, `engineering.example.com`, with their own certificate. Say the `finance` team wants `finance.example.com`, with a different certificate. This needs more than one Listener on the same `Gateway`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  gatewayClassName: internal-lb-class
  listeners:
    - name: engineering-https
      protocol: HTTPS
      port: 443
      hostname: "engineering.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: engineering-tls-secret
    - name: finance-https
      protocol: HTTPS
      port: 443
      hostname: "finance.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: finance-tls-secret
```

Notice both Listeners use the same `port` (443). That's allowed here, but only because each has a different `hostname`. This is where the rules in this guide start to matter, later on, we'll see exactly what makes two Listeners like this "distinct" versus "conflicting."

A `Gateway` can have more than one Listener. The maximum is 64.

**3. `addresses`**

Optional. Use it to request a specific IP. Otherwise, the controller assigns one for you.

Here's why this matters in practice. Say Chihiro's security team has a rule: every external entry point must have a fixed, known IP address. That IP goes into a firewall allowlist. It also goes into a DNS record.

Without `addresses`, `shared-gw` would get a random IP each time it's created. If Chihiro ever had to recreate it, the IP could change. The firewall rule and DNS record would break.

So Chihiro requests a specific IP instead:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  gatewayClassName: internal-lb-class
  addresses:
    - type: IPAddress
      value: "203.0.113.10"
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

Now `shared-gw` always uses `203.0.113.10`, even if it gets recreated later. The firewall rule and DNS record stay valid.

## TLS mode: a choice each Listener has to make

There's one field worth understanding before we go further. It's `tls.mode`. It changes how a Listener behaves. It has two possible values.

**1. Terminate mode (the default)**

In Terminate mode, the `Gateway` itself decrypts the traffic. To do this, it needs a certificate.

So the Listener must include a `certificateRefs` field. This field points to a Kubernetes Secret. The Secret must be of type `kubernetes.io/tls`.

Once the traffic is decrypted, the `Gateway` can read it. It can then hand the request off to an `HTTPRoute`.

This is the mode the `engineering` and `finance` teams would use for their websites.

**2. Passthrough mode**

In Passthrough mode, the `Gateway` never decrypts anything. It only reads the hostname visible in the TLS handshake, the SNI, from Part 1. Then it forwards the still-encrypted traffic straight to the backend.

The `Gateway` never touches the encrypted contents. So it doesn't need a certificate at all. `certificateRefs` isn't used here.

This is the mode `TLSRoute` has traditionally been paired with, to keep traffic encrypted end to end. (Newer Gateway API versions also allow `TLSRoute` to work with `Terminate` mode, but `Passthrough` remains the more common pairing, and the one this guide focuses on.) It's often used for things like databases, where the backend itself handles the encryption.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  gatewayClassName: internal-lb-class
  listeners:
    - name: engineering-https
      protocol: HTTPS
      port: 443
      hostname: "engineering.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: engineering-tls-secret
    - name: db-passthrough
      protocol: TLS
      port: 8443
      tls:
        mode: Passthrough
```

## The problem: what if two Listeners look identical to incoming traffic?

A `Gateway` with several Listeners needs to pick exactly one Listener for each piece of traffic. Say two Listeners are set up in a way where traffic could match either one. Now the `Gateway` can't decide. Kubernetes won't allow this.

Think of two doors on the same wall. Both have identical locks. Both open with the same key. Someone walks up with that key. Which door should open? There's no correct answer. So this setup simply isn't allowed.

Here's what this would actually look like for Chihiro. Say Chihiro sets up the `engineering-https` and `finance-https` Listeners from before, both on port 443, both HTTPS, but forgets to set a `hostname` on either one. Now incoming HTTPS traffic on port 443 could match `engineering-https`, or it could just as easily match `finance-https`. The `Gateway` has no way to tell them apart, no different door lock, so to speak. We'll see this exact scenario written out in the worked example below.

When two Listeners can't be told apart this way, they're called **Conflicted**. Conflicted Listeners can't exist together on the same Gateway.

## The distinctiveness rules

Different Listener types get compared differently. Why? Because each protocol shows a different amount of information. Some protocols show more. Some show less. So the rules can't be the same for all of them.

Before the rules, two quick definitions.

A **TLS Listener** has `protocol: TLS`. Its `tls.mode` can be set to either `Terminate` or `Passthrough`, the same choice described earlier in this guide. It pairs with `TLSRoute`.

An **HTTPS Listener** has `protocol: HTTPS`. Unlike a TLS Listener, it can only use `Terminate` mode, since `HTTPRoute` (which it pairs with) needs to read the decrypted request to do its job.

Now, the rules.

For TCP and UDP Listeners, only `protocol` and `port` matter. Say two Listeners are both on port 53. One is `TCP`. The other is `UDP`. These are distinct. The protocol is different, so there's no conflict. But say two Listeners are both `TCP`, both on port 22. These are not distinct. They conflict.

For TLS Listeners, three fields matter: `protocol`, `port`, and `hostname`. Two `TLS` Listeners can share the same port. They just need different hostnames.

For HTTPS Listeners, the same three fields apply: `protocol`, `port`, `hostname`. But there's one more rule. An HTTPS Listener must have a certificate. This is because HTTPS Listeners decrypt traffic, and decrypting needs a certificate. Different hostnames can use different certificates. But they don't have to, one certificate can often cover multiple hostnames.

## What actually happens when Listeners conflict

Say a `Gateway` ends up with Conflicted Listeners. Kubernetes doesn't ignore this quietly. The Listener's status gets marked `Conflicted: True`. Say none of the Listeners on a `Gateway` are usable as a result. Then the whole `Gateway` fails to reach an `Accepted` status.

One detail worth knowing: the controller can never pick a winner on its own. It can accept the rest of the Gateway's non-conflicting Listeners. It can leave the conflicting ones inactive. But it can't guess which one you "really meant."

## A worked example: giving Engineering and Finance their own domains

Let's continue the story. The `engineering` team wants `engineering.example.com`, with their own certificate. The `finance` team wants `finance.example.com`, with theirs. Both teams still keep their applications in their own namespaces, `engineering` and `finance`. Only the shared `Gateway`, `shared-gw`, lives in `infra`.

Their hostnames are different. So this is fine. These Listeners are distinct. Both can live on `shared-gw` at the same time:

```yaml
listeners:
  - name: engineering-https
    protocol: HTTPS
    port: 443
    hostname: "engineering.example.com"
    tls:
      mode: Terminate
      certificateRefs:
        - name: engineering-tls-secret
  - name: finance-https
    protocol: HTTPS
    port: 443
    hostname: "finance.example.com"
    tls:
      mode: Terminate
      certificateRefs:
        - name: finance-tls-secret
```

Now compare that to this. Here, Chihiro forgot to set the `hostname` field on either Listener:

```yaml
listeners:
  - name: engineering-https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
        - name: engineering-tls-secret
  - name: finance-https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
        - name: finance-tls-secret
```

Both Listeners now have the same `protocol`. Same `port`. No `hostname` at all. They're indistinguishable. This is a real conflict. `shared-gw` would fail to become `Accepted`, until Chihiro gives each Listener its own hostname.

**What happens if a user tries to visit the site anyway.**

1. A user opens `https://engineering.example.com` in their browser.
2. The request reaches `shared-gw`.
3. But `shared-gw` never became `Accepted`, because of the conflict. It was never properly set up.
4. There's nothing there to receive the request.
5. The user doesn't see a clear error message. The page just fails to load, like the address doesn't exist at all.

## A limit worth knowing about: 64 Listeners per Gateway

A single `Gateway` can hold a maximum of 64 Listeners. For most cases, this is more than enough. But large, multi-tenant setups can hit this ceiling. Imagine `shared-gw` eventually serving dozens of teams and namespaces, not just `engineering`, `finance`, and `hr`.

Gateway API has a newer resource for this. It's called `ListenerSet`. It lets Listeners be defined in separate objects. Those objects then get merged onto a `Gateway`. This is a newer, more advanced part of the API, and this series doesn't cover it in depth, but it's worth knowing it exists if `shared-gw` ever needs to scale beyond a handful of teams. The official docs, linked below, are the best place to go deeper on it.

## What comes next

Listeners, TLS modes, and distinctiveness rules should be clear now. The next guide moves to the Route side of things. It covers `GRPCRoute`, `TCPRoute`, and `UDPRoute` in more depth, along with the rules that decide how traffic gets matched once it reaches a Listener.

## Sources

- [API Overview, Gateway API official docs](https://gateway-api.sigs.k8s.io/docs/concepts/api-overview/)
- [Traffic Matching, Gateway API official docs](https://gateway-api.sigs.k8s.io/docs/concepts/traffic-matching/)
- [Gateway API v1.6: TCPRoute and UDPRoute Graduate to Standard, Kubernetes blog](https://kubernetes.io/blog/2026/08/03/gateway-api-v1-6-release/)
- [ListenerSet, Gateway API official docs](https://gateway-api.sigs.k8s.io/guides/user-guides/listener-set/)
