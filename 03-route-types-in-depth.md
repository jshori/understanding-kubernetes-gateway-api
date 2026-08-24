# Understanding Kubernetes Gateway API, Part 3: Route Types, In Depth

This is Part 3 of the "Understanding Kubernetes Gateway API" series. Part 1 introduced two Route types, `HTTPRoute` and `TLSRoute`. This part covers the other three: `GRPCRoute`, `TCPRoute`, and `UDPRoute`. It also gives you a simple way to decide which one to use.

## A recap: how much can each Route type actually see?

This is the one idea that ties all five Route types together. Each Route type can see a different amount of the traffic passing through it. Some can read a lot. Some can read almost nothing.

Think of it like reading a letter. Some Route types can open the envelope and read every word inside. Some can only read the name on the front of the envelope. Some can't even do that, they can only see which mailbox it arrived at.

## HTTPRoute: reads almost everything

`HTTPRoute` was covered in Part 1. It can read the URL path, the headers, the method, almost anything in an HTTP request. This is the most information any Route type can see.

## GRPCRoute: reads gRPC-specific details

gRPC is a way for services to talk to each other, often used between backend services rather than by end users directly. `GRPCRoute` is built specifically for it.

It can match on the hostname, on the specific gRPC service being called, on the specific method within that service, and on headers.

For example, the `engineering` team runs a gRPC-based service called `OrderService`, used internally by other services to fetch order details.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: order-service-route
  namespace: engineering
spec:
  parentRefs:
    - name: shared-gw
  hostnames:
    - "grpc.engineering.example.com"
  rules:
    - matches:
        - method:
            service: order.v1.OrderService
            method: GetOrder
      backendRefs:
        - name: order-cache-service
          port: 50051
```

For example, a request comes in for `grpc.engineering.example.com`, calling the `GetOrder` method on `OrderService`. Here's what happens: the `Gateway` matches the hostname first. Then `GRPCRoute` checks the specific service and method being called. Since both match this rule, the request goes to `order-cache-service`. If someone called a different method on the same service, say `CancelOrder`, this rule wouldn't match it, it would need its own rule, or a separate, more general rule to catch it.

One detail worth knowing: `GRPCRoute` needs an HTTPS Listener from Part 2 to attach to. This is because gRPC runs on top of HTTP/2, so it needs the same kind of decrypted, readable traffic that `HTTPRoute` needs.

## TCPRoute: reads almost nothing

`TCPRoute` is at the opposite end from `HTTPRoute`. It can't read hostnames. It can't read paths. It can't read headers. It only knows one thing: which port the traffic came in on.

For example, the `finance` team runs a database that other internal tools connect to directly, not over HTTP, just a raw database connection.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  gatewayClassName: internal-lb-class
  listeners:
    - name: finance-db
      protocol: TCP
      port: 5432
      allowedRoutes:
        kinds:
          - kind: TCPRoute
---
apiVersion: gateway.networking.k8s.io/v1
kind: TCPRoute
metadata:
  name: finance-db-route
  namespace: finance
spec:
  parentRefs:
    - name: shared-gw
      sectionName: finance-db
  rules:
    - backendRefs:
        - name: finance-db-service
          port: 5432
```

Notice there's no `hostnames` field here at all, and no `matches` field either. For example, a request comes in on port 5432. There's no hostname to check, and no path either, a raw database connection doesn't have either of those things. The `Gateway` looks at nothing except which port the connection arrived on, sees it's the `finance-db` Listener, and `TCPRoute` sends it straight to `finance-db-service`. That's the entire decision. This is the right tool for things like databases, where there's no concept of a hostname or path in the first place, just a raw connection.

## UDPRoute: the same idea, for UDP

`UDPRoute` works exactly like `TCPRoute`. The only difference is the protocol it matches, `UDP` instead of `TCP`. It's used for things like DNS servers, or other services that communicate over UDP instead of TCP.

For example, Chihiro's team also runs the company's internal DNS server, in the `infra` namespace, used by all three teams, `engineering`, `finance`, and `hr` alike, to resolve internal service names.

## A worked example: the DNS server needs both TCP and UDP

Here's a case where you genuinely need both `TCPRoute` and `UDPRoute` at the same time. DNS is unusual: it normally runs over UDP, but falls back to TCP for larger responses. Chihiro's DNS server needs to listen on port 53 for both protocols.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: dns-gateway
  namespace: infra
spec:
  gatewayClassName: internal-lb-class
  listeners:
    - name: dns-tcp
      protocol: TCP
      port: 53
      allowedRoutes:
        kinds:
          - kind: TCPRoute
    - name: dns-udp
      protocol: UDP
      port: 53
      allowedRoutes:
        kinds:
          - kind: UDPRoute
---
apiVersion: gateway.networking.k8s.io/v1
kind: TCPRoute
metadata:
  name: dns-tcp-route
  namespace: infra
spec:
  parentRefs:
    - name: dns-gateway
      sectionName: dns-tcp
  rules:
    - backendRefs:
        - name: dns-service
          port: 53
---
apiVersion: gateway.networking.k8s.io/v1
kind: UDPRoute
metadata:
  name: dns-udp-route
  namespace: infra
spec:
  parentRefs:
    - name: dns-gateway
      sectionName: dns-udp
  rules:
    - backendRefs:
        - name: dns-service
          port: 53
```

For example, `engineering`, `finance`, or `hr` sends a DNS query. Most of the time, it arrives over UDP, and the `UDPRoute` handles it. If the response is too large for UDP, the client retries over TCP instead, and the `TCPRoute` handles that. Both routes point to the same `dns-service`, they just handle different protocols on the same port.

Remember the distinctiveness rules from Part 2. These two Listeners are both on port 53, but one is `TCP` and the other is `UDP`. Different protocol, same port, no conflict.

## How to choose the right Route type

Ask what your traffic actually is, and how much of it needs to be readable to route it correctly.

If it's a normal web request, and you need to route by path, header, or hostname, use `HTTPRoute`.

If it's gRPC traffic, and you need to route by service or method, use `GRPCRoute`.

If it must stay encrypted end to end, and hostname alone is enough to route it, use `TLSRoute`, in Passthrough mode, from Part 2.

If it's raw TCP traffic with no concept of hostname or path, like a database, use `TCPRoute`.

If it's raw UDP traffic, like DNS, use `UDPRoute`.

## What comes next

The next guide moves away from Routes and back to conformance. It explains Release Channels, and the difference between Core, Extended, and Implementation-specific support, so you know which Gateway API features you can safely rely on across different controllers.

## Sources

- [gRPC routing, Gateway API official docs](https://gateway-api.sigs.k8s.io/guides/grpc-routing/)
- [GRPCRoute reference, Gateway API official docs](https://gateway-api.sigs.k8s.io/reference/api-types/grpcroute/)
- [Gateway API v1.6: TCPRoute and UDPRoute Graduate to Standard, Kubernetes blog](https://kubernetes.io/blog/2026/08/03/gateway-api-v1-6-release/)
- [GEP-2644: TCPRoute, Gateway API official docs](https://gateway-api.sigs.k8s.io/geps/gep-2644/)
- [API Reference, Gateway API official docs](https://gateway-api.sigs.k8s.io/reference/api-spec/main/spec/)
