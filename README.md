# Understanding Kubernetes Gateway API

A series that explains Kubernete's Gateway API from the ground up, one concept at a time, based directly on the official Gateway API documentation. Every technical term is explained the moment it comes up.

This series is meant for anyone who finds the official documentation too dense to start with. Nothing here is invented, every concept is sourced from the official docs (linked at the bottom of each part), just explained more slowly, with simple analogies and worked examples.

## Before you start

This series assumes basic familiarity with Kubernetes concepts like clusters, namespaces, and Services. It does not assume any prior knowledge of Gateway API, or of what an API is in general.

If you'd like a refresher on what an API is, or on the history of Ingress and why Gateway API was created, those are covered in a separate, related series: [kubernetes-ingress-and-gateway-api-fundamentals](https://github.com/jshori/kubernetes-ingress-and-gateway-api-fundamentals).

## Series contents

| Part | Title | Covers |
|---|---|---|
| 1 | [Roles, Personas, and the Resource Model](01-roles-personas-resource-model.md) | Who Gateway API is designed for (Ian, Chihiro, Ana), what `GatewayClass`, `Gateway`, `HTTPRoute`, and `TLSRoute` are, and how Routes can attach to Gateways across namespaces |
| 2 | [GatewayClass and Gateway, in depth](02-gatewayclass-and-gateway-in-depth.md) | Listener Distinctiveness, TLS modes (Terminate vs Passthrough), and the `addresses` field |
| 3 | [Route Types, in depth](03-route-types-in-depth.md) | `HTTPRoute`, `GRPCRoute`, `TLSRoute`, `TCPRoute`, `UDPRoute`, and how to decide which one to use |
| 4 | [ReferenceGrant and BackendTLSPolicy](04-referencegrant-and-backendtlspolicy.md) | Cross-namespace security, backend TLS validation |
| 5 | Service Mesh and GAMMA | *(coming soon)* How Gateway API extends to east-west (mesh) traffic |

## How to read this

Each part builds on the ones before it. New terms are always explained the first time they're used, so you shouldn't need to look anything up outside this series, but where it helps, each part links to the exact official documentation page the content is based on.
