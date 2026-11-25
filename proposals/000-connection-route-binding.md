<!--
Rename this file to proposals/<PR#>-connection-route-binding.md and update the title
to `# <PR#> - Connection-scoped routing: default destinations and route admission`
once the PR number is known (see proposals/README.md).
-->

# 000 - Connection-scoped routing: default destinations and route admission

Revise the `Router` SPI from [proposal 070][proposal-070] around two connection-scoped
concepts: a **default destination** that the runtime can forward to without consulting the
router, and an **admitted route** that the runtime enforces against the endpoint the client
actually connected to. `staticRoutes()` is removed. This proposal supersedes
[PR 118][pr-118] and [PR 156][pr-156], and closes a gap in the already-merged
[Subject Router proposal][proposal-140].

**TL;DR**

* A request has a runtime-defined **default target**. The router is consulted only when there
  is none. This restores the useful "forward to the endpoint the client connected to" default
  from before routing existed, without a per-API-key map.
* A router **admits** a connection for a route (`requireRoute(route)` or `acceptAnyRoute()`).
  On a node-bound connection the runtime checks the admitted route against the route the
  endpoint represents and rejects a mismatch itself. Routers no longer hand-write
  compare/reject/log/metric on every request to satisfy [PR 156][pr-156].
* Destination, route admission, and decode are separate questions. Decode remains a runtime
  decision, not a router declaration.
* The default target is `Optional` and runtime-defined, so the SPI does not assume that a
  `VirtualNode` is a physical broker. Alternative networking models remain possible.

## Current situation

[Proposal 070][proposal-070] introduced the `Router` plugin. As implemented today:

```java
interface Router {
    CompletionStage<RouterResponse> onRequest(ApiKeys apiKey, short apiVersion,
                                              RequestHeaderData header, ApiMessage request,
                                              RouterContext context);

    default Map<ApiKeys, String> staticRoutes() { return Map.of(); }

    default void close() {}
}
```

`staticRoutes()` maps API keys to fixed route names. Keys in the map are forwarded as opaque
frames to that route; every other key is decoded and handed to `onRequest()`. A `Router`
instance is per client connection; each router level has its own `NodeIdMapping`, and
`RouterContext` exposes `virtualNode()`, `anyNode(route)`, `nodeForId(int)`, `sendRequest()`,
`sessionId()`, `authenticatedSubject()` and the response builders.

[Issue #4177][issue-4177] and [PR 118][pr-118] identify three problems with `staticRoutes()`:

1. **The useful default was lost.** Before routing, traffic arriving on a broker endpoint
   always flowed to the broker that endpoint represented. `staticRoutes()` makes a router opt
   in to forwarding by listing API keys instead of inheriting that default.
2. **Scope mismatch.** `staticRoutes()` is a connection-independent map, but `Router` is
   per-connection. A route name cannot express "the node this connection is already bound to";
   that target is a `VirtualNode`, not a route.
3. **No router can populate the map usefully.** A pass-through router must enumerate ~80 keys;
   a client-id, subject or topic router has no API key that can be pinned to a route
   connection-independently.

[PR 156][pr-156] identifies a separate problem. A client chooses which broker-specific
endpoint to connect to, and that endpoint may belong to any route in the topology.
`RouterContext.virtualNode()` returns that node, but a router cannot tell which route it
belongs to. A malicious client can therefore connect to a node endpoint on a route it should
not reach and rely on the router forwarding there. The same information would support checks
such as verifying that a `PRODUCE` arrived on the node that leads the topic-partition
([#4606][issue-4606]).

The [Subject Router proposal][proposal-140] is the first production-shaped router and already
resolves its route per request. It forwards to `virtualNode()` when the client is on a
broker-specific endpoint and to `anyNode(route)` on bootstrap. Its text assumes that "those
connections resolve back through the same route because the subject, and therefore the route,
is unchanged". A client that connects to a different route's endpoint breaks that assumption:
the router will forward data-plane traffic to the endpoint the client chose, not the route its
subject maps to. [PR 156][pr-156] therefore describes a real gap in 140, not only a
hypothetical one.

The proof of concept in [kroxylicious#4510][pr-4510] implements 118's `shouldIntercept()` and
demonstrates that a bound/unbound default is workable. It also surfaced pre-existing
out-of-band-request defects in `main` that are independent of this proposal.

## Motivation

Seven requirements drive this revision:

1. **Restore default forwarding**, without enumerating API keys. A request that the router
   does not care about should go where the connection already points.
2. **Make the default a runtime fact, not a topology assumption.** "Forward to the assigned
   node" is only meaningful while a virtual node maps to exactly one broker. The SPI must not
   bake that assumption in, because 070 deliberately left room for proxy-as-broker and shared
   mappings where it is false.
3. **Make route confinement enforceable and hard to forget.** Per-request
   `query → compare → reject → metric → log` in every router is boilerplate, is easy to omit,
   and runs against requirement 1's desire to avoid per-request routing work.
4. **Keep decode a runtime decision.** Rob's point in #4177 stands: whether to deserialise
   depends on filters, routers, and the runtime's node-ID translation. It is not the router's
   declaration to make.
5. **Keep the change small and sympathetic.** `onRequest()`, `sendRequest()`, the response
   builders, `virtualNode()`, `anyNode()`, `nodeForId()`, `sessionId()` and
   `authenticatedSubject()` should not change. Routers that work today should need the
   smallest possible edit.
6. **Fail closed.** A connection must not be default-forwarded on the basis of an endpoint the
   router has not admitted.
7. **Remain stateless across proxy instances.** Admission must be re-derivable on the instance
   that terminates a connection; no cross-instance communication or shared admission state
   should be required.

## Proposal

### Design principles

* **Destination** — the runtime can often say where a request goes without the router. That is
  the *default target*. The router chooses a destination only when there is no default, or
  when it deliberately intercepts.
* **Admission** — the router says which route(s) a connection is allowed to use. The runtime
  enforces that the connection's endpoint belongs to an admitted route. This is the security
  boundary, and it is connection-scoped, not per-request.
* **Decode** — the runtime decodes if a filter, the router, or node-ID translation needs the
  body. The SPI exposes no decode-depth declaration.
* **The router is trusted; the runtime owns the enforcement.** A router may explicitly
  `sendRequest()` to any node it likes (for fan-out, topology discovery, and so on). The
  runtime's job is to stop a *client* reaching a route the router did not admit, not to sandbox
  the router.

### Concepts

**Default target.** The `VirtualNode` the runtime will forward a request to if the router
declines to intercept it. It is `Optional` because there may be no single destination:
bootstrap connections (unless the runtime can derive exactly one), connections whose router has
not yet admitted a route, API keys the runtime cannot forward transparently, and future
topologies where forwarding is content-dependent.

**Assigned route.** The route that the connection's endpoint represents *at this router level*.
It is empty on bootstrap, and empty when the endpoint is not tied to exactly one route (for
example a future shared mapping). It is expressed in this router level's own route namespace,
so nested routers each see their own value.

**Admission.** A connection-scoped declaration by the router that the connection may use a
route. Admission is what makes the default target available. Once admitted, the router is not
consulted for requests that have a default target; the runtime forwards them. Admission is
immutable for the life of the connection in this proposal (see
[Re-authentication](#re-authentication)).

### The SPI

`Router` loses `staticRoutes()` and gains one method:

```java
public interface Router {

    /**
     * Whether the runtime must consult this router for this request.
     *
     * <p>Default: consult the router only when there is no default target for the request.
     * When this returns false the runtime forwards the request unchanged to
     * {@link RouterContext#defaultTarget()}.
     *
     * <p>Implementations must not return false when {@code defaultTarget()} is empty; the
     * runtime treats that as a programming error and closes the connection.
     *
     * <p>This method must be side-effect free. A router may be asked more than once for the
     * same request.
     */
    default boolean shouldIntercept(ApiKeys apiKey, short apiVersion, RouterContext context) {
        return context.defaultTarget().isEmpty();
    }

    CompletionStage<RouterResponse> onRequest(ApiKeys apiKey, short apiVersion,
                                              RequestHeaderData header, ApiMessage request,
                                              RouterContext context);

    default void close() {}
}
```

`RouterContext` keeps every existing method unchanged and gains four:

```java
public interface RouterContext {

    // ... existing methods unchanged ...

    /**
     * The node this request will be forwarded to if shouldIntercept() returns false, or empty
     * if there is no default destination.
     *
     * <p>Empty on bootstrap until the router admits a single route, on a node-bound connection
     * the router has not yet admitted, and for API keys the runtime cannot forward
     * transparently.
     *
     * <p>Becomes available as soon as the router admits the connection for a route.
     */
    Optional<VirtualNode> defaultTarget();

    /**
     * The route this connection's endpoint represents at this router level, or empty on
     * bootstrap or where the endpoint does not represent exactly one route.
     *
     * <p>Available immediately, before admission, so a router can answer API_VERSIONS for the
     * particular broker the client addressed without waiting for authentication.
     */
    Optional<String> assignedRoute();

    /**
     * Admits this connection for the given route and only that route.
     *
     * <p>On a node-bound connection the runtime immediately compares {@code route} with
     * {@link #assignedRoute()}. If they differ, or if the endpoint does not represent exactly
     * one route, the runtime rejects the connection: it logs at WARN, increments a metric, and
     * closes the connection. The router does not have to write that check itself.
     *
     * <p>On bootstrap, admission records the route the connection is expected to use and makes
     * the default target available.
     *
     * <p>Admission is sticky. Re-declaring the same route is a no-op. Declaring a different
     * route throws {@link RouteAdmissionException}; the runtime treats a propagating exception
     * as an admission failure and closes the connection. A router that wants a Kafka-level
     * error rather than a close may catch the exception and use
     * {@link #respondWithError}.
     *
     * <p>Admission is local to the proxy instance that terminates the connection and is not
     * shared with other instances. Route selection must be re-derivable on every instance from
     * the connection's own inputs and the router's configuration; see
     * <em>Multiple proxy instances and statelessness</em>.
     */
    void requireRoute(String route);

    /**
     * Admits this connection for whatever route its endpoint represents. For routers whose
     * connections legitimately span routes, such as a union-cluster router handling METADATA
     * on a data connection.
     *
     * <p>On bootstrap this admits the connection without a default target: there is no single
     * route to forward to, so the router continues to be consulted.
     */
    void acceptAnyRoute();
}
```

`RouteAdmissionException` is a new unchecked exception in
`io.kroxylicious.proxy.router`. It denotes a router admitting a route that the connection's
endpoint does not represent, or changing an existing admission.

### Lifecycle and dispatch

The runtime keeps the admission state on the connection (alongside the existing
`NodeIdMapping` and endpoint node ID), not on the per-request `RouterContext`. A sketch:

```java
class ConnectionBinding {
    final Optional<Integer> endpointNodeId;   // present for a node-bound connection
    final Optional<String> assignedRoute;     // derived from endpointNodeId, this level
    Optional<String> admittedRoute;           // set by requireRoute
    boolean admittedAnyRoute;                 // set by acceptAnyRoute

    boolean admitted() { return admittedAnyRoute || admittedRoute.isPresent(); }

    Optional<VirtualNode> defaultTarget(ApiKeys key, RouterContextImpl ctx) {
        if (!admitted()) {
            return Optional.empty();
        }
        if (endpointNodeId.isEmpty()) {
            // Bootstrap: only a single admitted route gives a destination.
            return admittedRoute.map(ctx::anyNode);
        }
        // Bound: forward to the endpoint's node, but only for APIs the runtime
        // can forward transparently.
        return ctx.canForwardTransparently(key) ? ctx.virtualNode() : Optional.empty();
    }
}
```

Per request:

```java
var ctx = new RouterContextImpl(frame, dispatcher, binding, sessionId, subject);
if (!router.shouldIntercept(apiKey, apiVersion, ctx)) {
    var target = ctx.defaultTarget()
            .orElseThrow(() -> new IllegalStateException(
                    "Router declined to intercept but there is no default target"));
    forwardUnchanged(frame, target);
} else {
    dispatchToRouter(apiKey, apiVersion, frame, ctx);
}
```

The `shouldIntercept()` gate is OR'd into the runtime's decode predicate, exactly as in
118:

```java
boolean shouldDecodeRequest(ApiKeys key, short version) {
    return filterDecodePredicate.shouldDecodeRequest(key, version)
            || router.shouldIntercept(key, version, routerContext);
}
```

`NodeIdResponseTranslator` and the runtime's node-ID-translation set are unchanged. They
operate whether or not the router is consulted.

### Request shapes

The three shapes from 118 survive, but admission is now explicit:

| shape | admission | gate | `onRequest` does |
|---|---|---|---|
| **pass-through** | `requireRoute(assignedRoute or sole route)` once | `shouldIntercept == false` after admission | not called; runtime forwards to `defaultTarget()` |
| **pinned** | `requireRoute(route)` | `shouldIntercept == true` for the pinned keys | `sendRequest` to one fixed route/node |
| **fan-out** | `acceptAnyRoute()` | `shouldIntercept == true` for cluster-spanning keys | decompose, send to each route, recompose |

### Router examples

**Static / pass-through.** Fronts one cluster and never inspects requests. It admits on the
first request it sees; after that the runtime forwards everything.

```java
class StaticRouter implements Router {
    @Override
    public CompletionStage<RouterResponse> onRequest(ApiKeys key, short version,
            RequestHeaderData header, ApiMessage request, RouterContext ctx) {
        ctx.requireRoute(ctx.assignedRoute().orElse(SOLE_ROUTE));
        return ctx.sendRequest(ctx.defaultTarget().orElseThrow(), header, request)
                .thenApply(r -> ctx.respondWith(r).build());
    }
}
```

**Client-id.** Chooses a route from the client-id in the request header. It admits on the
first request it sees (the default gate keeps consulting it until it does); bound connections
are then validated once and pass through.

```java
class ClientIdRouter implements Router {
    @Override
    public CompletionStage<RouterResponse> onRequest(ApiKeys key, short version,
            RequestHeaderData header, ApiMessage request, RouterContext ctx) {
        ctx.requireRoute(routeFor(header.clientId()));
        return ctx.sendRequest(ctx.defaultTarget().orElseThrow(), header, request)
                .thenApply(r -> ctx.respondWith(r).build());
    }
}
```

No `shouldIntercept()` override is needed: the default gate consults the router until it has
admitted the connection, and forwards afterwards.

**Subject.** The [Subject Router proposal][proposal-140] resolves its route per request and
forwards to `virtualNode()` or `anyNode(route)`. With this proposal it instead admits once the
subject is known (the example omits 140's handling of pre-authentication `API_VERSIONS`):

```java
class SubjectRouter implements Router {
    @Override
    public CompletionStage<RouterResponse> onRequest(ApiKeys key, short version,
            RequestHeaderData header, ApiMessage request, RouterContext ctx) {
        String route = routeForSubject(ctx.authenticatedSubject());
        ctx.requireRoute(route);                 // validates the bound endpoint's route
        return ctx.sendRequest(ctx.defaultTarget().orElseThrow(), header, request)
                .thenApply(r -> ctx.respondWith(r).build());
    }
}
```

On a bound connection whose endpoint belongs to another route, `requireRoute` fails and the
runtime closes the connection. That is exactly the check 156 asks every router to write by
hand, and it now also fixes 140's assumption.

**Topic / union.** A connection may legitimately touch several routes, so it admits any route
and still fans out for cluster-spanning APIs.

```java
class TopicRouter implements Router {
    @Override
    public boolean shouldIntercept(ApiKeys key, short version, RouterContext ctx) {
        return ctx.defaultTarget().isEmpty() || spansClusters(key) || pinned(key);
    }

    @Override
    public CompletionStage<RouterResponse> onRequest(ApiKeys key, short version,
            RequestHeaderData header, ApiMessage request, RouterContext ctx) {
        ctx.acceptAnyRoute();
        // ... decompose, send via ctx.sendRequest(ctx.anyNode(route), ...), recompose ...
    }
}
```

The `ctx.defaultTarget().isEmpty()` term is the "consult until admitted" gate; after admission
it is false, so only the cluster-spanning and pinned keys are intercepted. An `isAdmitted()`
convenience could make this read better (see [Open questions](#open-questions)).

### Decode

This proposal changes only the destination/admission gate. It does not add
`shouldDecodeRequest()` to `Router`, because decode is the runtime's decision: a request must
be decoded if any filter, the router, or node-ID translation needs it.

Two consequences worth stating explicitly:

* **Interception currently implies a body decode.** `onRequest` receives an `ApiMessage`, so a
  router that intercepts is decoded. Avoiding that for header-only routers is follow-on work
  and is not specific to this proposal.
* **Response decoding is unchanged.** Node-ID translation already decodes the
  node-bearing responses, including `FETCH`; 118's investigation concluded that lazy response
  decoding would save too little to justify widening `sendRequest`.

### Nested routers

Each router level has its own `assignedRoute()`, admission state, route names and
`NodeIdMapping`. Admission at one level does not admit the connection at another; each router
decides for itself, in its own namespace. The runtime composes the per-level bindings exactly
as it composes the existing node-ID mappings. A nested router that is not admitted is
consulted, just like a top-level one.

### Forwarding unknown APIs

One of Tom's objections to 118 was that open-by-default silently forwards a future API key
whose responses carry node IDs, so the runtime fails to translate them. The answer here is
that the runtime, not the router, owns "can this be forwarded transparently?". `defaultTarget()`
is empty for any API key the runtime does not recognise as safe to forward without
consultation, so the router is consulted and the case is visible rather than silent. If the
router also cannot handle the key, it can reject with `UNSUPPORTED_VERSION` or close. This is a
runtime safety policy and does not reintroduce a router-populated key map.

### Re-authentication

Kafka clients can re-authenticate on an existing connection (KIP-368), and filters may change
the `Subject` for other reasons. Admission is therefore bound to the subject observed when it
was made. Before consulting the router, the runtime compares the current subject with the
subject recorded at admission. If it differs, the runtime clears admission and consults the
router again; the router re-admits the same route (a no-op) if the new identity still maps
there.

A different route on an already-admitted connection throws `RouteAdmissionException` and the
connection is closed. That matches [proposal 140][proposal-140]'s fail-closed handling of a
mid-connection identity change: the connection carries cluster-specific state (cached virtual
node IDs, in-flight requests, transaction, group and fetch-session state) that cannot be
migrated, so the client must reconnect. If a future use case needs to move a connection between
routes that always lead to the same cluster, admission can be relaxed to allow it.

### API version negotiation

`API_VERSIONS` is scoped to a single broker connection. On a bound connection,
`assignedRoute()` is available before admission, so a router can answer `API_VERSIONS` from
`virtualNode()` — the exact broker the client addressed, including its version range during a
rolling upgrade — even while the subject is still anonymous. That connection will be rejected
at its first data-plane request if the authenticated subject maps to another route, so no
data-plane traffic reaches the wrong cluster.

On bootstrap there is no assigned route, so the router is consulted. Once it admits a route it
can answer from `anyNode(route)`; until then, a subject router that must support an unknown
eventual route (for example, SASL-terminated clients that negotiate before authenticating) can
still intersect versions across routes as [proposal 140][proposal-140] describes.

### Failure handling and observability

An admission failure is a security-relevant event, so the runtime handles it uniformly:

* log at WARN with virtual cluster, session ID, router name, assigned route and admitted route;
* increment a per-route `route_admission_rejected` counter;
* close the connection (the client is either malicious or broken; closing is consistent with
  the project's fail-closed guidance).

A router that wants protocol-level control may catch `RouteAdmissionException` and return
`respondWithError(header, request, error, message).andCloseConnection().build()`.

Existing per-route request counters and latency histograms are unchanged. The runtime can now
attribute default-forwarded requests to the admitted route without the router having to call
`sendRequest`, which also removes a source of missing route attribution in metrics.

### Security model

* **The client is untrusted.** It can connect to any node endpoint the proxy exposes and can
  put any node ID in a request. It can choose the endpoint, but it cannot make the router admit
  a route other than the one that endpoint represents; the router decides admission from the
  connection's identity, and the runtime enforces the match.
* **The router is trusted code.** It is the policy decision point. The runtime does not
  sandbox explicit `sendRequest()` calls, because a router must be able to fan out and to
  perform discovery. The runtime's guarantee is narrower and more useful: a *client* is never
  default-forwarded on a connection the router has not admitted.
* **Admission is the boundary.** Before admission there is no default target, so every request
  is either handled by the router or rejected by it. After admission, default forwarding only
  goes to the node the endpoint represents.
* **Pre-authentication API_VERSIONS.** On a bound connection a router may answer
  `API_VERSIONS` from `virtualNode()` before the subject is known. That reveals version
  information about the broker the client already addressed, and no data-plane request is
  forwarded before admission. If the subject later maps to a different route, the first
  data-plane request is rejected at admission.
* **Route names are per router level.** `assignedRoute()` and `requireRoute()` use the calling
  router's own route namespace, so a nested router cannot accidentally admit against its
  parent's route names.

### Multiple proxy instances and statelessness

Kroxylicious is commonly deployed as several proxy instances (for example a Kubernetes
`Deployment`), with no communication between them. Clients open connections to one or more
instances believing them to be distinct brokers, and a client's connections may land on
different instances. The design must not require an admission made on one instance to be known
to another; it does not.

The key distinction is between the **client-visible virtual node** and the **connection-scoped
admission**:

* A virtual node ID is a deterministic function of the topology configuration (`V = id + S × t`
  in the dedicated mapping). Every instance computes the same `(route, target broker)` for the
  same virtual node. This is existing 070 behaviour, and is why route `id`s are explicit and
  `S` must be consistent across instances.
* An admission is not a property of the client or the virtual cluster. It is a property of one
  TCP connection, made by the instance that terminates it. Every new connection is admitted
  independently.

No admission state is shared, and none needs to be. The bootstrap connection that admitted a
client to route R tells the other instances nothing. When the client opens a broker-specific
connection, that instance derives the endpoint's virtual node, recomputes the client's identity
from that connection's own authentication, and admits the connection itself.

Making admission **connection-scoped**, rather than a client- or session-level property, is
what makes this work without cross-instance state. Kafka has no session identity spanning
connections that the proxy could use; a client simply opens independent TCP connections to the
brokers it learned from `METADATA`. A client-scoped admission would therefore require the
instances to agree on a session identity and share the decision, which is exactly the
cross-proxy communication this design avoids.

This remains safe even when instances disagree, because `requireRoute` is validated against the
endpoint's assigned route. If instance A would admit a client to R and instance B's policy
would admit it to S, B still sees an endpoint for route R and rejects the connection; it cannot
forward an R-addressed connection to S. Policy divergence therefore manifests as a failed
connection (fail-closed), not as cross-route leakage.

The model relies on the same invariants as 070:

1. **Deterministic node mapping.** All instances present the same virtual node IDs for the same
   `(route, target broker)` and resolve them back identically.
2. **The endpoint identifies the node at every instance.** A broker-specific connection must
   carry enough information (for example a per-node port or SNI) for the terminating instance
   to derive the same virtual node. If it does not, `assignedRoute()` is empty and the
   connection is not default-forwarded; the router must handle it or reject. A load balancer
   that hides the virtual node from the proxy is not a supported default-forwarding
   deployment.
3. **Identical router configuration and policy.** Router configuration, route names and ids,
   and any subject/client-id mapping must be deployed consistently. Coordinated reconfiguration
   and draining are already required by 070.
4. **Per-connection identity.** Each instance must be able to establish the client's identity
   on the connection it terminates. Kafka clients re-authenticate per connection under SASL,
   and mTLS presents the certificate per connection; an identity established only on one
   connection cannot be used to admit another.
5. **Route selection is re-derivable.** Route selection must be a pure function of the
   connection's own inputs (assigned route, authenticated subject, request headers) and static
   configuration, or consult externally shared state with a documented consistency model. A
   router that chooses a route from per-connection state that is not re-derivable on another
   instance (a random choice, local round-robin, an unsynchronised cache) will diverge across
   instances and is not suitable for a multi-instance deployment. Topology-dependent choices
   (leader, replica, coordinator) are node-targeting decisions *within* an admitted route and
   may legitimately differ per instance; they must not change the admitted route itself.

This also assumes the proxy terminates authentication on each connection. SASL passthrough,
where the proxy does not learn the client's identity, cannot support identity-based admission;
that is consistent with [proposal 140][proposal-140]'s treatment of SASL frames.

`anyNode(route)` is deliberately not stable across instances: different instances may choose
different brokers on the same route. Only the route is guaranteed. For bootstrap requests, and
for any request that is not broker-specific, this is the intended behaviour; the client learns
the actual nodes from `METADATA`.

The Subject Router example is multi-instance-safe: each instance independently authenticates
the client and computes the same route from the same mapping, even if the client's connections
land on different instances.

### Implementation notes

* Add a connection-scoped admission holder next to the existing endpoint node ID in
  `RoutingHandler`/`RoutingHandler.nested()`, and pass it to each per-request
  `RouterContextImpl`.
* `RouterContextImpl.assignedRoute()` is
  `nodeIdMapping.fromVirtual(endpointNodeId).route()` when the level has an endpoint node, and
  empty otherwise. It needs no new derivation.
* `RouterContextImpl.defaultTarget()` is computed from the binding as sketched above; it must
  consider the current API key so the runtime can decline transparent forwarding.
* `requireRoute()` validates against `assignedRoute()`, records the admission, and throws
  `RouteAdmissionException` on mismatch or change. `acceptAnyRoute()` records the opt-out.
* `KafkaProxyInitializer.computeDecodedKeysForRouter()` is replaced by the per-connection
  predicate that ORs `shouldIntercept()` into filter decode interest, as 118 already sketches.
  `NODE_ID_TRANSLATION_APIS` remains unconditional. The predicate is evaluated before a
  per-request context exists, so it needs a lightweight connection-scoped view (endpoint
  binding, admission, subject), not a freshly allocated `RouterContextImpl`.
* Clear admission when the `Subject` changes and re-consult the router on the next request.
* The PoC in [kroxylicious#4510][pr-4510] is a useful starting point for the dispatch changes;
  its out-of-band-request fixes are independent and should be tracked separately.

## Relationship to other proposals

* **[Proposal 070][proposal-070]** — this amends its `Router`/`RouterContext` SPI. Concepts,
  routes, the DAG, node-ID mapping, `TopologyService`, response builders and configuration are
  unchanged.
* **[PR 118][pr-118]** — superseded. It correctly identified the `staticRoutes()` problems and
  the `shouldIntercept` gate; this proposal keeps the gate but replaces the bootstrap-only
  default with a runtime-defined `defaultTarget()` and adds admission.
* **[PR 156][pr-156]** — satisfied, without adding `routeOf(VirtualNode)`. `assignedRoute()`
  exposes the connection's route, and `requireRoute()` moves the compare/reject/log/metric
  into the runtime. For [#4606][issue-4606]'s leader-affinity check, compare
  `virtualNode()` with `TopologyService.leaderOf(topic, partition)` directly; exact node
  equality is stronger than route equality and needs no route string.
* **[Proposal 140][proposal-140]** — should be updated to admit its route instead of
  forwarding on every request. That both closes its bound-endpoint gap and enables the
  "collapse to a fixed forwarding path once the route is established" optimisation it already
  lists as future work. Its per-request `API_VERSIONS` fan-out on bound connections can also be
  replaced by answering from `virtualNode()`.

## Affected/not affected projects

* **`kroxylicious-api`** — `Router.staticRoutes()` removed; `Router.shouldIntercept()` added;
  `RouterContext` gains `defaultTarget()`, `assignedRoute()`, `requireRoute()`,
  `acceptAnyRoute()`; new `RouteAdmissionException`.
* **`kroxylicious-runtime`** — connection-scoped admission state; dispatch consults the
  default target; decode predicate ORs in `shouldIntercept()`; admission metrics and logging;
  subject-change invalidation. The existing node-ID translation and nested-router machinery
  are unchanged.
* **Router plugin authors** — `staticRoutes()` implementors migrate to the default gate; the
  no-op edit is usually to admit the assigned route on the first request. A router that never
  admits is still correct, but is consulted per request.
* **`kroxylicious-integration-tests`** — test routers migrate; add coverage for admission
  success/failure, nested routers, and re-authentication.
* **Not affected:** the `Filter` API, KMS, authoriser API, existing configurations. The
  operator/CRDs need no SPI change, but the deployment/network model must keep the
  endpoint-to-virtual-node binding resolvable on every proxy instance, as 070 already
  requires.

## Compatibility

* `070` routing is unreleased/behind a feature flag, so removing `staticRoutes()` has no user
  impact (118 already relies on this).
* The change to `RouterContext` is additive except for `staticRoutes()`. Existing method
  signatures and semantics are retained.
* A router written against 118's `shouldIntercept` compiles, but its default (bootstrap-only)
  must be revisited because `defaultTarget()` now also depends on admission. That is a
  deliberate behavioural change: the safe default is "consult until admitted", not "forward on
  any bound connection".
* Shared/many-to-one node-ID mappings remain out of scope for default forwarding:
  `assignedRoute()` is empty, `defaultTarget()` is empty, and the router is consulted. This
  preserves 070's room for alternative networking models instead of assuming a 1:1
  virtual-node-to-broker mapping.

## Rejected alternatives

* **Keep `staticRoutes()`.** No router can populate it usefully, and a route name cannot name
  the node a connection is bound to.
* **118 as written: `shouldIntercept` defaulting to `virtualNode().isEmpty()`.** It leaves
  156's bound-endpoint hole open for every router that does not hand-write the check, and it
  assumes that a bound virtual node always has a single upstream broker, which forecloses the
  proxy-as-broker model.
* **`routeOf(VirtualNode)` on its own (156's proposal).** It exposes topology to routers and
  still leaves enforcement to every router: query, compare, reject, log, metric, and remember
  to do it on every request. `assignedRoute()` plus `requireRoute()` answers the same question
  once, in the runtime.
* **Auto-admit the assigned route on connection creation.** It closes 156's hole only for
  routers that are allowed to reach every route. A subject or tenant router must reject an
  endpoint that does not match its identity, and an automatic admission would silently bypass
  that policy. Explicit admission is the safer default; routers that do span routes call
  `acceptAnyRoute()`.
* **A sealed `RouteAction` instead of a boolean.** `FORWARD_TO_DEFAULT`, `FORWARD_TO_NODE`,
  `DISPATCH`, `MULTICAST` would leave room for explicit forwarding without decode and for
  multicast. No current router needs it, and the boolean keeps symmetry with the filter API's
  `shouldHandle*` predicates. It can be added additively later; `defaultTarget()` is already
  the concept a `FORWARD_TO_DEFAULT` action would need.
* **A `RoutingMode` enum (`PASS_THROUGH`, `HEADER_ONLY`, `FULL_MESSAGE`).** It conflates
  destination with decode depth. `PASS_THROUGH` has no destination on bootstrap, so the
  connection gate is needed anyway; decode depth is a separate (deferred) concern.
* **Per-request route-confinement checks in the router.** This is 156's original shape. It is
  easy to forget, produces inconsistent error codes and metrics across routers, and adds
  per-request work to the path 118 is trying to make cheap.
* **A shared subject→route cache enforced by the runtime.** It would require cross-connection
  state, would not handle re-authentication or routers whose connections legitimately span
  routes, and would move router policy into the runtime. Admission stays connection-scoped and
  router-owned.
* **Sharing admission or session state across proxy instances.** A cross-instance store would
  let an admission made on one instance be reused on another, but it requires communication or
  a shared dependency, contradicts the stateless deployment model, and adds a consistency
  problem to a security decision. It is unnecessary because admission is re-derivable per
  connection from the endpoint binding and the client's per-connection identity; when it is
  not re-derivable, failing closed is preferable to reusing a stale decision.
* **Header-only interception and progressive decoding in this proposal.** Real, but separable
  and blocked on the generated `RequestData`/`ResponseData` types; deferred.
* **Lazy response decoding.** Already investigated and rejected in 118: node-ID translation
  decodes the node-bearing responses anyway, leaving too narrow a case to justify widening
  `sendRequest`.

## Open questions

1. **Naming.** `requireRoute` / `acceptAnyRoute` are proposed. Alternatives: `commitToRoute`,
   `admitRoute`, `admitAnyRoute`. The review on 156 used "commit to a route"; consistency with
   that discussion may be worth more than the security flavour of "admit".
2. **Admission failure semantics.** Close by default (proposed) versus always synthesise an
   API-specific error. Closing is simpler and consistent with fail-closed; routers that need a
   particular error can catch `RouteAdmissionException`.
3. **Whether admission may ever change.** Proposed immutable, which matches 140's
   fail-closed-on-subject-change. Allowing a move between routes that resolve to the same
   cluster would need a runtime notion of cluster identity and is deferred.
4. **`isAdmitted()` convenience.** A small accessor equivalent to
   `defaultTarget().isEmpty()` would make "intercept until admitted" gates read better; decide
   whether it earns a place on the SPI.
5. **Should `acceptAnyRoute()` be a distinct method or `requireRoute(null)`/`requireAnyRoute()`.**
   A distinct method makes the opt-out explicit and auditable, which seems worth the extra
   name.
6. **Sealed action type.** If explicit-forward-without-decode or multicast becomes a real
   requirement, replace the boolean gate with a sealed `RoutingDecision` and deprecate
   `shouldIntercept`. Nothing in this proposal prevents that.
7. **Declaring multi-instance determinism.** The runtime cannot prove that a router's route
   selection is deterministic across instances. A factory-level declaration (or a documented
   requirement) could make the constraint explicit and visible to operators; is that worth
   adding to `RouterFactoryContext`?

## References

[proposal-070]: ./070-routing-api.md
[proposal-140]: ./140-subject-router.md
[pr-118]: https://github.com/kroxylicious/design/pull/118
[pr-156]: https://github.com/kroxylicious/design/pull/156
[pr-4510]: https://github.com/kroxylicious/kroxylicious/pull/4510
[issue-4177]: https://github.com/kroxylicious/kroxylicious/issues/4177
[issue-4606]: https://github.com/kroxylicious/kroxylicious/issues/4606

