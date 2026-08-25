# Understanding Kubernetes Gateway API, Part 4: ReferenceGrant and BackendTLSPolicy

This is Part 4 of the "Understanding Kubernetes Gateway API" series. This part covers two resources focused on security: `ReferenceGrant`, which controls a different kind of cross-namespace access than the one covered in Part 1, and `BackendTLSPolicy`, which handles the other side of a TLS connection we haven't looked at yet.

## A different kind of cross-namespace question

Part 1 covered one cross-namespace scenario: a Route attaching to a Gateway in a different namespace, using the handshake between `allowedRoutes` on the Gateway and `parentRefs` on the Route. That's how `engineering`, `finance`, and `hr` were each able to attach their own Route to Chihiro's shared `shared-gw`, even though `shared-gw` lived in the `infra` namespace.

There's a separate scenario Part 1 didn't cover. Let's say the `finance` team's Route doesn't just want to attach to the shared Gateway, it also wants to send traffic to a Service that lives somewhere else entirely, in a different namespace. Or let's say Chihiro wants the `Gateway`'s TLS certificate to come from a Secret that a separate security team manages, stored in their own namespace, not in `infra`.

Neither of these is about attaching a Route to a Gateway, that part is already solved. These are about referencing a completely different kind of resource, a Service or a Secret, across a namespace boundary. This is what `ReferenceGrant` handles, and it's a separate mechanism from the one in Part 1.

## ReferenceGrant: permission to reference something in another namespace

By default, a Route can only send traffic to a Service in its own namespace. A Gateway can only use a Secret in its own namespace. If you try to reference something across a namespace boundary without permission, it's simply rejected, similar in spirit to the default we saw in Part 1, just applied to a different pair of resources.

`ReferenceGrant` grants that permission. It works in a specific direction: it's created in the namespace of the resource being referenced (the target), and it lists which namespace, and which kind of resource, is allowed to reference it.

For example, the `finance` team's `HTTPRoute` needs to send traffic to a shared caching service that the platform team runs centrally, in a `platform` namespace, instead of duplicating that service inside every team's own namespace. Without a `ReferenceGrant`, this reference would be rejected.

The platform team creates the grant, inside their own `platform` namespace:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: ReferenceGrant
metadata:
  name: allow-finance-to-cache
  namespace: platform
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: finance
  to:
    - group: ""
      kind: Service
```

Then the `finance` team's `HTTPRoute` can reference the Service directly:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: finance-app-route
  namespace: finance
spec:
  parentRefs:
    - name: shared-gw
      namespace: infra
  rules:
    - backendRefs:
        - name: shared-cache-service
          namespace: platform
          port: 6379
```

Notice the `backendRefs` entry explicitly names the target `namespace`, the same pattern as `parentRefs` did back in Part 1.

*(A quick note on this example: the specific scenario, a finance team, a platform namespace, a shared cache, is one we've built for this series to make the mechanism concrete. The underlying mechanism itself, that `ReferenceGrant` is required for cross-namespace Route-to-Service and Gateway-to-Secret references, and that it's separate from the Route-to-Gateway handshake in Part 1, is drawn directly from the official Gateway API documentation.)*

## Why this exists: a real security risk

Think of the `Gateway`'s controller as a mailroom worker with a master key to every office in a building. If someone hands this worker a package labeled "deliver to the CEO's office," the worker just delivers it, no questions asked, since the worker has access everywhere.

Now imagine someone from the mailroom itself, or a low-level intern with no real access, writes that label. They can't get into the CEO's office themselves. But the mailroom worker can. So they trick the mailroom worker into doing it for them.

This is exactly the risk with cross-namespace references. The `Gateway`'s controller is like that mailroom worker, it has the technical ability to reach resources across many namespaces. Without `ReferenceGrant`, anyone who can create a Route could quietly point it at a sensitive Service in a namespace they'd never normally be allowed to touch, and the controller would just do it, since it never checks whether that's actually okay.

`ReferenceGrant` fixes this by adding one rule: the controller only delivers the "package" if the target's owner has explicitly said, in advance, "yes, this specific sender is allowed to send things to me."

## The same idea applies to Gateways and Secrets

`ReferenceGrant` isn't limited to Routes and Services. It works the same way if a `Gateway` in one namespace needs to use a TLS certificate Secret stored in a different namespace, the team owning that Secret would create a `ReferenceGrant` allowing `Gateway` resources from the `Gateway`'s namespace to reference Secrets in theirs.

## BackendTLSPolicy: the other end of the TLS connection

Recall Terminate mode from Part 2. In that mode, the `Gateway` decrypts traffic coming in from the client, using a certificate defined in `certificateRefs` on the Listener. That solved one side of the connection, the client talking to the `Gateway`.

But there's a second side we haven't covered: what happens between the `Gateway` and the actual backend Pod? If that backend Pod also expects HTTPS, not plain HTTP, the `Gateway` needs to trust the certificate the backend presents. `BackendTLSPolicy` is what configures that.

Unlike a Route, `BackendTLSPolicy` attaches to a **Service**, not a Route. Here's why that matters.

Let's say the `engineering`, `finance`, and `hr` teams each have their own `HTTPRoute`, and all three happen to send some of their traffic to the same shared caching service in the `platform` namespace:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: engineering-cache-route
  namespace: engineering
spec:
  parentRefs:
    - name: shared-gw
      namespace: infra
  rules:
    - backendRefs:
        - name: shared-cache-service
          namespace: platform
          port: 6379
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: finance-cache-route
  namespace: finance
spec:
  parentRefs:
    - name: shared-gw
      namespace: infra
  rules:
    - backendRefs:
        - name: shared-cache-service
          namespace: platform
          port: 6379
```

(The `hr` team's Route would look identical, just in its own namespace.)

If `BackendTLSPolicy` attached to each Route instead of the Service, all three teams would need to write, and keep updating, the exact same TLS settings, one copy per Route. Miss one, and that team's connection to the cache breaks.

By attaching one `BackendTLSPolicy` to the Service instead, it's written once, in the `platform` namespace, by the team that owns the cache:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: BackendTLSPolicy
metadata:
  name: cache-backend-tls
  namespace: platform
spec:
  targetRefs:
    - group: ""
      kind: Service
      name: shared-cache-service
  validation:
    hostname: shared-cache-service.platform.svc.cluster.local
    wellKnownCACertificates: System
```

Now all three Routes, `engineering`, `finance`, and `hr`, automatically validate the cache's certificate the same way, without any of them needing to configure anything themselves.

Two fields are required here. `hostname` is the SNI the `Gateway` presents when connecting to the backend, and it must match what's on the backend's own certificate. Then either `caCertificateRefs` (pointing to a specific certificate to trust) or `wellKnownCACertificates: System` (trust the same well-known public certificate authorities your browser already trusts) must be set, not both.

Here's what this looks like in practice, once this policy exists. When any Route (`engineering`, `finance`, or `hr`) sends traffic to `shared-cache-service`, the `Gateway` opens a TLS connection to that Pod, and during the handshake, it presents `shared-cache-service.platform.svc.cluster.local` as the hostname it's trying to reach. The backend Pod responds with its certificate. The `Gateway` checks two things: does the certificate's own hostname match what was just presented, and was that certificate signed by a CA the `Gateway` trusts (which is what `wellKnownCACertificates: System` or `caCertificateRefs` controls). Only if both checks pass does the `Gateway` consider the backend trustworthy and forward the traffic.

Both `ReferenceGrant` and `BackendTLSPolicy` deal with a similar question, can something in one namespace reach into another namespace? But they answer it very differently, and it's worth being clear about that difference so you don't assume one always behaves like the other.

`BackendTLSPolicy` has a rule: it must sit in the exact same namespace as the Service it's protecting. If the Service is in `platform`, the policy must also be in `platform`. There's no way around this, no field exists anywhere in `BackendTLSPolicy` that lets you point to a Service in a different namespace.

`ReferenceGrant`, by contrast, is built entirely around crossing namespaces. It has a `from` field specifically for naming a different namespace and letting it in.

`BackendTLSPolicy` simply doesn't have anything like that `from` field. There's nothing to configure for cross-namespace access, because the option doesn't exist in the first place.

## Putting both TLS pieces together

Between Part 2 and this part, there are now two separate TLS decisions in play, and it's worth being clear they're independent of each other. A full request actually passes through both, one after the other.

Take the `engineering` team's traffic as an example. Recall from Part 2 that `shared-gw` has a Listener for `engineering.example.com`, in Terminate mode, using the `engineering-tls-secret` certificate. When a client sends a request to `engineering.example.com`, the `Gateway` decrypts it right there, using that certificate. That's the first decision, the client-to-Gateway side, covered entirely by Terminate mode from Part 2.

Now say that request needs data from the shared cache in the `platform` namespace, the one we just added `BackendTLSPolicy` for. The `Gateway` now has to make a second, completely separate decision: when it forwards this request to `shared-cache-service`, should that connection also be encrypted, and can it trust the certificate the cache presents back? That's the second decision, the Gateway-to-backend side, and it's what `BackendTLSPolicy` from this part configures.

Notice these are two different certificates, doing two different jobs. `engineering-tls-secret` proves the `Gateway`'s identity to the client. The cache's own certificate, validated by `BackendTLSPolicy`, proves the cache's identity to the `Gateway`. One request, from client to `engineering.example.com`, all the way to the cache, passes through both checks, decrypted once coming in, then re-encrypted and verified again going out.

## What comes next

The final guide in this series moves from Gateway API's original purpose, north-south traffic from Part 1, into Service Mesh territory: east-west traffic between services inside the cluster, and the GAMMA initiative that made this possible.

## Sources

- [ReferenceGrant, Gateway API official docs](https://gateway-api.sigs.k8s.io/reference/api-types/referencegrant/)
- [GEP-709: Cross Namespace References from Routes, Gateway API official docs](https://gateway-api.sigs.k8s.io/geps/gep-709/)
- [BackendTLSPolicy, Gateway API official docs](https://gateway-api.sigs.k8s.io/reference/api-types/policy/backendtlspolicy/)
- [GEP-1897: BackendTLSPolicy, Gateway API official docs](https://gateway-api.sigs.k8s.io/geps/gep-1897/)
- [TLS Configuration, Gateway API official docs](https://gateway-api.sigs.k8s.io/guides/tls/)
