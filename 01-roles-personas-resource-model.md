# Understanding Kubernetes Gateway API, Part 1: Roles, Personas, and the Resource Model

This is Part 1 of a series that explains Kubernete's Gateway API from the ground up, one concept at a time. Every technical term is explained as it comes up, in plain English, so no prior Gateway API knowledge is assumed. This part covers who Gateway API is designed for, and the three main types of objects you'll work with throughout the rest of this series.

*(This guide assumes you already know what a Kubernetes cluster is, and that a cluster can be divided into separate sections called namespaces, used to keep different teams' resources organized and separated from each other. If any of that is new to you, it's worth looking up a basic Kubernetes overview first.)*

## The problem: shared infrastructure, different people

Imagine a big apartment building. There's the **building owner** (who owns the land, the pipes, the electrical wiring), the **building manager** (who decides which floor gets what, handles maintenance requests, sets rules for the building), and the **tenants** (who just want to live in their apartment without worrying about plumbing or electrical work).

All three care about the building, but they care about **different parts of it**, and they shouldn't need to step on each other's toes to get their job done. This exact situation happens with Kubernetes infrastructure too, and it's the core problem Gateway API was designed to solve.

According to the official Gateway API docs: *"In practice, clusters and their infrastructure tend to be shared, which the original Ingress model doesn't capture very well. A critical factor is that when infrastructure is shared, not everyone using the infrastructure has the same concerns, and to be successful, an infrastructure project needs to address the needs of all the users."*

## The three official personas

Gateway API's design is built around three specific "characters" (yes, they officially have names and even pronouns, defined in the docs):

### Ian, the Infrastructure Provider

Ian's job is "the care and feeding of a set of infrastructure that permits multiple isolated clusters to serve multiple tenants." He doesn't care about any single tenant specifically, he cares about the infrastructure working well for everyone collectively. In the real world, **Ian will often work for a cloud provider (AWS, Azure, GCP) or a PaaS provider.**

Think of Ian as the **building owner**: he owns the actual foundation, the wiring, the load balancers.

### Chihiro, the Cluster Operator

Chihiro's job is to manage a single cluster, making sure it meets the needs of all its users. They're "typically concerned with policies, network access, application permissions, etc." Chihiro isn't loyal to any one team either, they need the cluster to work for everyone using it.

Think of Chihiro as the **building manager**: they decide who can use which floor, and set the rules.

### Ana, the Application Developer

Ana is different from the other two in an important way: her focus is on **her application's business needs**, not on Kubernetes or Gateway API itself. In fact, per the official docs, "Ana is likely to view Gateway API and Kubernetes as pure friction getting in her way to get things done." She just wants her app to work and be reachable.

Think of Ana as the **tenant**: she just wants her apartment to have working internet, she doesn't want to think about the building's electrical wiring.

## An important real-world note

In a small startup, one person might actually be Ana *and* Chihiro at the same time, with Ian being an automated cloud provider process in the background. In a large company, these could easily be three completely separate teams who rarely talk to each other directly. Gateway API's design has to work for both extremes.

## How this connects to actual Kubernetes permissions (RBAC)

This isn't just a nice story, it maps directly onto real access control. Kubernetes uses **RBAC (Role-Based Access Control)** to decide who is allowed to create, view, or modify which resources. The official docs state: *"We anticipate that each persona will map approximately to a Role in the Kubernetes Role-Based Authentication (RBAC) system."*

In practice, this means a real Kubernetes cluster could be set up so that only Ian's team can create `GatewayClass` objects, only Chihiro's team can create `Gateway` objects, and Ana's team can create Route objects, but nothing above that. This is enforced with actual Kubernetes permissions, not just convention.

## The three main resource types, at a glance

According to the official docs, there are three main types of objects in the Gateway API resource model:

| Resource | Who typically owns it |
|---|---|
| **`GatewayClass`** | Ian (infrastructure provider) |
| **`Gateway`** | Chihiro (cluster operator) |
| **Routes** (`HTTPRoute`, `TLSRoute`, etc.) | Ana (application developer) |

Now that we know who owns each of these, let's actually look at what each one is, in detail.

## What is a GatewayClass?

A `GatewayClass` is a **template**. It doesn't create anything by itself. It just says "here is a type of gateway that can be built."

One important detail: a `GatewayClass` is **cluster-scoped**. This means it isn't tied to one specific namespace, it applies to the whole cluster, and any namespace can use it.

Think of a car dealership's showroom. The showroom has a sign that says "We build Sedans" and another sign that says "We build SUVs." Those signs are just templates. No actual car has been built yet. You're only seeing what's possible.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: my-gateway-class
spec:
  controllerName: example.com/gateway-controller
```

## What is a Gateway?

A `Gateway` is where things get real. When you create a `Gateway`, you're saying "actually build me one, using that template."

Once you create it, the controller notices it, and does real work. It creates an actual, working entry point, usually with a real IP address or hostname that traffic can reach.

Going back to the car analogy: this is you walking into the dealership and saying "I want one Sedan, please build it for me." Now a real car exists.

A `Gateway` needs to say exactly what kind of traffic it will accept. It does this using **listeners**. Each listener is one rule: "accept traffic on this port, using this protocol."

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: my-gateway-class
  listeners:
    - name: http
      protocol: HTTP
      port: 80
```

## What is a Route?

Once traffic arrives at a `Gateway`, something needs to decide where that traffic should actually go next, which service inside the cluster should handle it. That's the job of a **Route**.

"Route" isn't one single resource. It's a family of resources, one for each kind of traffic. `HTTPRoute` handles HTTP traffic. `TLSRoute` handles raw, encrypted TLS traffic. There are others too, but these two are the most common, and they also help explain an important idea, so we'll focus on them.

Every Route, no matter which kind, needs two things: which `Gateway` it wants to attach to (this field is called `parentRefs`), and where to send the matching traffic (this is inside `rules`, using a field called `backendRefs`).

### HTTPRoute

`HTTPRoute` can look inside a request. It can read things like the URL path, or specific headers, and make decisions based on what it sees. For example: "if the path is `/api`, send it here, otherwise send it there."

But here's the catch. To read inside a request like that, the request has to already be readable. It can't still be encrypted. So `HTTPRoute` only works after the `Gateway` has already decrypted the traffic.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-http-route
spec:
  parentRefs:
    - name: my-gateway
  hostnames:
    - "example.com"
  rules:
    - matches:
        - path:
            value: /api
      backendRefs:
        - name: api-service
          port: 8080
```

### TLSRoute

`TLSRoute` is for the opposite situation, when you want to keep the traffic encrypted all the way to the backend, and never let the `Gateway` read it at all.

If the `Gateway` can't read the encrypted traffic, how does it know where to send it? It uses the one piece of information that's visible even before encryption fully kicks in, the **hostname** the client is trying to reach. This is why `TLSRoute` always requires you to specify a hostname, there's no other way for it to make a decision.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: TLSRoute
metadata:
  name: my-tls-route
spec:
  parentRefs:
    - name: my-gateway
      sectionName: tls-passthrough
  hostnames:
    - "db.example.com"
  rules:
    - backendRefs:
        - name: database-service
          port: 5432
```

## A simple way to remember the difference

Ask yourself one question: **does the Gateway need to read the traffic, or just pass it along untouched?**

If it needs to read it (to check paths or headers), use `HTTPRoute`. If it should stay sealed and untouched all the way to the backend, use `TLSRoute`.

## A question this naturally raises

We said `Gateway` belongs to Chihiro, and Routes belong to Ana. But Chihiro and Ana are different people, often on completely different teams, in different namespaces. Does that mean their resources have to live together in the same namespace?

Not necessarily, and understanding how this works explains how one shared entry point can serve many independent teams at once.

## The default: namespaces are isolated from each other

By default, a `Gateway` will only accept Routes that live in its **own namespace**. If a `Gateway` exists in a namespace called `infra`, and someone creates an `HTTPRoute` in a completely different namespace called `engineering` that tries to attach to it, **this will be rejected by default**. This is intentional, it's a safety measure so that no namespace can silently attach itself to a Gateway it wasn't invited to use.

## How cross-namespace attachment actually works: a handshake

The official docs describe this using a very fitting word: **a handshake**. For a Route in one namespace to successfully attach to a Gateway in a different namespace, **both sides have to explicitly agree**:

**Side 1, permission from the Gateway's owner:** Chihiro has to explicitly configure the Gateway's listener to allow Routes from other namespaces, using a field called `allowedRoutes.namespaces`. This field can be set to:
- **`Same`** (the default): only Routes from the Gateway's own namespace
- **`All`**: Routes from any namespace are allowed to attach
- **`Selector`**: only Routes from namespaces that carry a specific label are allowed

**Side 2, an explicit reference from the Route:** The Route has to name the exact Gateway (and its namespace) it wants to attach to, in its `parentRefs` field.

If only one side agrees, nothing happens. Chihiro allowing "All" namespaces is meaningless if no Route ever references that Gateway, and a Route trying to reference a Gateway that hasn't allowed it will simply be rejected. Both halves of the handshake are required.

## A worked example: Engineering, Finance, and HR

Let's make this concrete with a scenario that's easy to picture: a company with three application teams, Engineering, Finance, and HR, each with their applications running in their own namespace (`engineering`, `finance`, `hr`). None of these namespaces need to know anything about networking, TLS, or load balancers, that's Chihiro's job. So Chihiro creates one dedicated namespace, `infra`, purely to hold the shared Gateway.

First, Chihiro creates the Gateway, and explicitly opens the door to other namespaces:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gw
  namespace: infra
spec:
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      allowedRoutes:
        namespaces:
          from: All   # explicitly allowing Routes from any namespace
```

Then, each team creates their own Route, in their own namespace, explicitly naming the Gateway they want to use:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: engineering-route
  namespace: engineering
spec:
  parentRefs:
    - name: shared-gw
      namespace: infra   # explicitly naming which Gateway, and which namespace it's in
```

Finance and HR would create an almost identical `HTTPRoute`, just in their own namespace, pointing to the same `shared-gw`. None of the three teams need to know that the other two exist. Chihiro only had to build and secure one entry point, instead of three separate ones. This is the practical, everyday value of the persona-based design we started this guide with: Chihiro keeps control over the shared infrastructure, while Ana (in each team) keeps full control over her own application's routing.

## Confirming this against the official documentation

This exact pattern, one shared Gateway, multiple independent teams attaching Routes to it, is not something we invented for this example. It's described directly in the official Gateway API documentation, using a similar setup with three teams named A, B, and C:

> Chihiro has deployed a Gateway `shared-gw` in the `infra` Namespace, to be used by different application teams for exposing their applications outside the cluster. Team A and Team B (in Namespaces `A` and `B`) attach their Routes to this shared Gateway. They are unaware of each other, and as long as their Route rules don't conflict, they can keep operating in isolation. Team C has special needs (performance, security, or criticality) and needs a **dedicated** Gateway. Team C deploys their own Gateway `dedicated-gw` in Namespace `C`, usable only by apps in that namespace.

This adds one more useful detail beyond our Engineering/Finance/HR example: Team C shows that sharing isn't mandatory. If a team has a good reason (say, stricter security requirements, or the need for a completely separate TLS certificate), they're free to deploy their own `Gateway` inside their own namespace instead. In that case, there's no cross-namespace handshake needed at all, because the `Gateway` and the `Route` both simply live in the same namespace together, the default case we described earlier.

Gateway API doesn't force every team into one model. It supports both: shared, for teams happy to pool infrastructure, and dedicated, for teams that need their own.

## What comes next

Now that we understand *who* owns which piece, what each resource actually is, and how Routes and Gateways can connect across namespace boundaries, the next guide will dig deeper into `GatewayClass` and `Gateway`, including something called **Listener Distinctiveness**, a set of rules that decide whether two Listeners on the same Gateway are allowed to coexist or whether they conflict with each other.

## Sources

- [Introduction, Gateway API official docs](https://gateway-api.sigs.k8s.io/docs/introduction/)
- [Roles and Personas, Gateway API official docs](https://gateway-api.sigs.k8s.io/docs/concepts/roles-and-personas/)
- [API Overview, Gateway API official docs](https://gateway-api.sigs.k8s.io/docs/concepts/api-overview/)
- [Security, Gateway API official docs](https://gateway-api.sigs.k8s.io/docs/concepts/security/)
- [Cross-Namespace routing, Gateway API official docs](https://gateway-api.sigs.k8s.io/guides/multiple-ns/)
- [TLSRoute reference, Gateway API official docs](https://gateway-api.sigs.k8s.io/reference/api-types/tlsroute/)
