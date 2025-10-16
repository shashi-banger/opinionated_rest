
# REST and Hypermedia API Design Guide

## Table of Contents

1. [HATEOAS](#hateoas)
2. [RESTful Checklist](#restful-checklist)
3. [REST as Mediator Pattern](#rest-as-mediator-pattern)
4. [Factory Pattern in REST](#factory-pattern-in-rest)
5. [Workflow Modeling](#workflow-modeling)
6. [SIREN Hypermedia Format](#siren-hypermedia-format)
7. [RFC Standards](#rfc-standards)
8. [ALPS (Application-Level Profile Semantics)](#alps-application-level-profile-semantics)

---

## HATEOAS

With HATEOAS, a client interacts with a network application whose application servers provide information dynamically through hypermedia. A REST client needs little to no prior knowledge about how to interact with an application or server beyond a generic understanding of hypermedia.

Fielding insisted on **Hypermedia as the Engine of Application State (HATEOAS)**:

- The client should discover actions via links embedded in responses, not hardcoded rules.
- This is like a mediator dynamically guiding the client, saying: "Here's what you can do next."
- **Interaction through hypermedia not code!**

**Reference:** [HATEOAS on Wikipedia](https://en.wikipedia.org/wiki/HATEOAS)

## RESTful Checklist

Here's a tight, no-nonsense RESTful API Checklist (Fielding-aligned) you can pin in a design doc or use in reviews.

### A. Resource Modeling

- **Nouns, not verbs, in URIs.**
  - ✅ `/orders/123` ❌ `/orders/123/cancel`, `/createOrder`
- **Stable identifiers.** URIs identify resources; they don't encode actions or workflows.
- **Collection vs. item.**
  - Collection: `/orders`
  - Item: `/orders/{id}`
- **Sub-resources only when they're real resources:** `/orders/{id}/items`
### B. Uniform Interface (HTTP Semantics)

- **Use HTTP methods for state transitions.**
  - `GET` = retrieve (safe)
  - `POST` = create / server-initiated processing on a collection or controller resource
  - `PUT` = full replace (idempotent)
  - `PATCH` = partial update (state change)
  - `DELETE` = delete (idempotent)
- **No RPC verbs in paths.** Express intent via method + representation (e.g., `PATCH /orders/123 { "status": "cancelled" }`).
### C. Hypermedia (HATEOAS)

- **Responses advertise next actions via links.**

**Example (HAL/JSON-ish):**
```json
{
  "id": 123,
  "status": "pending",
  "_links": {
    "self": { "href": "/orders/123" },
    "cancel": {
      "href": "/orders/123",
      "method": "PATCH",
      "body": { "status": "cancelled" }
    }
  }
}
```

- **Clients discover affordances at runtime instead of hardcoding workflows.**
### D. Self-Descriptive Messages

- **Content negotiation:** Accept, Content-Type (JSON, HTML, etc.).
- **State carried in representations.** Server doesn't require out-of-band knowledge to interpret requests.
- **Validation & errors:** Use problem details (`application/problem+json`) with clear type, title, detail.

### E. Statelessness & Layered System

- **No session server state.** Each request is complete on its own (auth tokens, correlation IDs, etc.).
- **Layer-friendly.** Works through proxies, gateways, CDNs without breaking semantics.

### F. Cacheability

- **Use HTTP caching headers.** ETag/If-None-Match, Cache-Control, Last-Modified/If-Modified-Since.
- **Safe methods cacheable by default** (GET, HEAD).

### G. Idempotency & Safety

- **GET/HEAD safe.** No side effects.
- **PUT/DELETE idempotent.** Repeating the call leaves state the same.
- **PATCH not guaranteed idempotent**—design operations carefully.

### H. Status Codes (clear, consistent)

- **2xx:** 200 OK, 201 Created (with Location), 204 No Content.
- **3xx:** 304 Not Modified, appropriate redirects if needed.
- **4xx:** 400/401/403/404/409/422.
- **5xx:** Server faults.
- **Don't overload 200 for all outcomes.**

### I. Versioning & Evolution

- **Prefer evolvability over hard versioning in URIs.**
- **If versioning is required,** consider media type versioning (e.g., `Accept: application/vnd.example.v2+json`).
- **Keep URIs stable;** evolve representations and link relations.

### J. Pagination, Sorting, Filtering

- **Represented as hypermedia controls.** Provide next, prev, first, last links; include server-acknowledged params in the response.
- **Don't rely on undocumented query semantics.**

### K. Security (at the protocol boundary)

- **Auth via standard headers** (OAuth2/OIDC bearer tokens in Authorization).
- **Don't leak implementation details;** keep error messages useful but not revealing.
### Quick "REST or Not?" Sniff Tests
- **Verb-in-URI test:** Any action word after a resource ID? (e.g., `/cancel`, `/approve`) → Not REST.
- **Affordance test:** Can a client discover what to do next from response links alone? If not → Not RESTful enough.
- **Method semantics test:** If you change state with GET → Wrong. If you "create" with PUT to a server-assigned ID → Wrong.
- **Idempotency test:** Repeating PUT/DELETE should be safe. If not → Wrong.
- **Cache test:** Can intermediaries cache safe responses with ETags? If not considered → Weak REST.
- **Self-descriptive test:** Could another team integrate with just the HTTP spec + your responses? If they need a PDF of bespoke rules → Not REST.
### Anti-Patterns → RESTful Rewrites
- ❌ `POST /orders/123/cancel`
  - ✅ `PATCH /orders/123 { "status": "cancelled" }` + link relation cancel
- ❌ `POST /createOrder`
  - ✅ `POST /orders` (server assigns ID; return 201 Created + Location)
- ❌ `GET /runDailyBilling`
  - ✅ Model a resource or asynchronous job: `POST /billing-cycles` → 202 Accepted + job resource with status links
- ❌ Session-bound server state
  - ✅ JWT/OAuth tokens; every request self-contained
### Micro-Rubric (score yourself fast)
Resources & URIs (0–2)
HTTP method semantics (0–2)
Hypermedia/HATEOAS (0–2)
Self-descriptive messages & negotiation (0–2)
Caching & idempotency (0–2)
 16–20: Strong REST 10–15: Serviceable but improve HATEOAS/caching
 <10: Likely RPC-over-HTTP
### HATEOAS Example

**Reference:** [HATEOAS on Wikipedia](https://en.wikipedia.org/wiki/HATEOAS)

```http
GET /accounts/12345 HTTP/1.1
Host: bank.example.com

HTTP/1.1 200 OK

{
    "account": {
        "account_number": 12345,
        "balance": {
            "currency": "usd",
            "value": 100.00
        },
        "links": {
            "deposits": "/accounts/12345/deposits",
            "withdrawals": "/accounts/12345/withdrawals",
            "transfers": "/accounts/12345/transfers",
            "close-requests": "/accounts/12345/close-requests"
        }
    }
}
```

> `/accounts/12345/withdrawals` is a list! Presented as a hyperlink, not constructed through code! 

## REST as Mediator Pattern

RESTfulness is very close to being the mediator pattern, just at Web scale.

### 1. Mediator Pattern: Core Idea

In the classic mediator pattern, objects don't talk directly. Instead, they send messages or intents to a mediator, which coordinates interactions and keeps things loosely coupled.

### 2. REST as a Mediator

Roy Fielding's REST defines constraints that make the REST interface itself the mediator between clients (UIs) and servers (resources, services, databases).

- **Client → Mediator (REST API):** The client doesn't call backend functions, it expresses desired state changes via HTTP methods (GET, PUT, PATCH, DELETE).
- **Mediator → Server:** The REST API translates those requests into internal operations (database queries, business logic, messaging).
- **Mediator → Client:** The API returns representations of resource state, not implementation details.

So: the RESTful interface is a mediator that hides backend complexity and enforces uniform communication.

### 3. REST Constraints as Mediator Properties

Each REST constraint echoes mediator behavior:

- **Statelessness** → Each interaction is a self-contained message to the mediator. No hidden coupling via server sessions.
- **Uniform interface** → Just like a mediator defines a common protocol for objects, REST enforces HTTP verbs + resource URIs.
- **HATEOAS** → The mediator doesn't just forward; it guides the client dynamically, telling it what transitions are possible.
- **Layered system** → Clients don't know if they're talking to the origin server, a cache, or a proxy — the mediator abstracts it all away.

### 4. Why This Matters

Without REST's mediator role, clients would bind directly to backend operations (RPC). That leads to tight coupling, brittle APIs, and no evolvability.

With REST, the mediator ensures loose coupling, evolvability, scalability, and interoperability.

✅ **So the comment holds true:**
> RESTfulness is the Web's embodiment of the mediator pattern — it mediates between clients and backend internals by enforcing resource-based, state-driven interactions through a uniform contract.
### Mediator Pattern vs. RESTfulness

In the mediator pattern, objects do not communicate directly. Each participant sends its messages to the mediator, which interprets them and decides how to propagate or transform them. This keeps the participants decoupled from one another.

In REST, the client (UI) and the backend internals are likewise decoupled. They never talk directly. Instead, they interact through the RESTful interface — resources exposed via URIs, manipulated with HTTP verbs. This interface plays the role of the mediator.

**Key Parallels:**

- **Central protocol:** In the mediator pattern, there's a fixed set of ways to talk to the mediator. In REST, this is the uniform interface: GET, POST, PUT, PATCH, DELETE, and standard representations.
- **State transitions:** In mediator style, the mediator defines how an object's request translates into a change in another object. In REST, the mediator ensures that state transitions happen through resource representations, not through procedural calls.
- **Decoupling:** In mediator style, objects don't know about each other's internal logic. In REST, the client doesn't know if the order cancellation is implemented by refunding a payment, sending an email, or updating a database — the only thing it sees is the change in resource state.
- **Discoverability:** In mediator style, the mediator can guide what interactions are possible. In REST, this appears as HATEOAS — the server tells the client what next transitions are available.
- **Layering:** In mediator style, the mediator can introduce layers of logic, routing, or orchestration. In REST, the layered system constraint means the client can't tell if it is talking to the origin server, a proxy, or a gateway. That indirection is another form of mediation.
- **Stateless messages:** In the mediator pattern, communication is message-based. In REST, each request is self-contained and stateless, which makes the mediator's job clear and repeatable.

### The Essence

REST takes the essence of the mediator pattern — loose coupling, centralized coordination, clear protocol — and applies it to the Web as a distributed system. That is why it scales: every client and server can evolve independently as long as they speak the language of the mediator, i.e. the REST constraints.
### A Story: REST as the Mediator Pattern

Here's a short story that dramatizes REST as the mediator pattern.

**The Cast:**
- **Client (UI):** A web app that wants to place and manage orders.
- **Mediator (REST interface):** The API, speaking HTTP and resource representations.
- **Backend (Server internals):** Databases, business logic, payment systems.

**Scene 1 — The Client Speaks**
- **Client:** "I'd like to see what order 123 looks like."
- **Mediator:** "Sure. Use GET /orders/123. I'll return a JSON representation."
- **Mediator (to Backend):** "Hey, fetch me the state of order 123."
- **Backend:** "Here it is: status=pending, items=[…]."
- **Mediator (to Client):** "Here's the resource representation."

> **Notice:** The client didn't ask the backend directly, didn't care about database tables. The mediator handled the translation.

**Scene 2 — A State Change**
- **Client:** "I want to cancel this order."
- **Mediator:** "That means changing the order's state. Send me a PATCH to /orders/123 with { status: 'cancelled' }."
- **Mediator (to Backend):** "Perform the necessary operations: refund, update records, log the action."
- **Backend:** "Done."
- **Mediator (to Client):** "Order 123 is now cancelled."

> **Again:** The client didn't call cancelOrder() or /cancel. It simply described a new state. The mediator coordinated all backend effects.

**Scene 3 — Guidance via Hypermedia**
- **Mediator (to Client):** "By the way, since this order is cancelled, you can't pay it anymore. But you could reorder it — here's a link for that."

> Here the mediator doesn't just relay; it guides. The client discovers next steps dynamically, not from hardcoded logic.

**Scene 4 — Layers at Work**
- **Client:** "I'm talking to /orders/123, but are you the real server?"
- **Mediator:** "Doesn't matter. I might be a cache, a proxy, or the origin. The contract is the same. You'll never know — and you don't need to."

> The mediator abstracts the entire layered system away.

**Epilogue:**
- The client never invoked backend methods.
- The backend never cared about UI quirks.
- The mediator (REST) enforced uniform rules, mediated state changes, and preserved loose coupling.

That is why we can say: **RESTfulness is the mediator pattern, applied to the Web.**



### The RPC World (For Comparison)

**The Cast (RPC World):**
- **Client (UI):** Still the web app.
- **Backend (Server):** Exposes functions over HTTP, more like remote procedures than resources.

**Scene 1 — Fetching Data**
- **Client:** "I want order 123."
- **Backend:** "Call my function getOrderById at /getOrder?orderId=123."
- **Client:** "Okay, I'll remember that function name and query parameter forever."
- **Backend:** "Here's the response. But if I rename the function or change parameters, you'll break."

> Coupling is already tight — the client is tied to server's procedure naming.

**Scene 2 — Cancelling the Order**
- **Client:** "I want to cancel this order."
- **Backend:** "Call /cancelOrder?id=123. That's my cancellation function."
- **Client:** "Alright, but I have to know that cancelOrder exists as a verb, not just a state change."
- **Backend:** "Exactly. And if I later decide it should be /voidOrder, you'll need to update your client."

> Instead of a uniform contract (PATCHing state), the client must know the backend's vocabulary of verbs.

**Scene 3 — Discoverability**
- **Client:** "How do I know what else I can do with this order?"
- **Backend:** "Well… read the docs. I don't embed next actions in my responses."
- **Client:** "So I can't discover capabilities dynamically?"
- **Backend:** "No, you must hardcode workflows against my API spec."

> The client and backend are tightly coupled again — changes require coordination.

**Scene 4 — Layers**
- **Client:** "Can I go through a cache or proxy?"
- **Backend:** "Not really, because my endpoints don't follow uniform semantics. A proxy can't just optimize them."

> RPC breaks the layered abstraction — caches can't make sense of /cancelOrder.

**Epilogue (RPC World):**
- Every call is a direct procedure invocation.
- URIs leak backend function names.
- Clients and servers are tightly coupled — evolution is painful.
- No mediation, no uniformity, no discoverability.

### Contrast to REST

In REST, the mediator (uniform interface + resources + hypermedia) shields both sides. In RPC, the client and backend stare directly at each other, with every change on one side rippling painfully into the other.

**Key Principle:** "REST is about state mediation, RPC is about verb invocation"



## Factory Pattern in REST

A "factory" can be modeled in a RESTful way. You don't expose verbs like `/createWidget`; you use resource-oriented creation with POST, guided by hypermedia. Here's how to realize common factory needs while staying inside Fielding's constraints.

### 1) The Collection-as-Factory (the usual, RESTful way)

A collection resource acts as your factory. Clients POST a representation to the collection; the server (your "factory logic") decides how to instantiate the new thing.

```http
POST /widgets
Content-Type: application/json

{ "kind": "gear", "diameter": 25, "material": "steel" }
```

Server behavior (factory work) is internal. The REST contract:
- 201 Created on success, with Location of the new resource.
- The response body is the representation of the created resource.
- The client never calls a "makeGear()" method; it just POSTs to /widgets.

```http
HTTP/1.1 201 Created
Location: /widgets/91f…c2
Content-Type: application/json

{ "id": "91f…c2", "kind": "gear", "diameter": 25, "material": "steel", ... }
```
### 2) Polymorphic/Abstract Factory (multiple product types)

You've got variants (gear, spring, shaft). Several RESTful options:

- **Type field in the representation** (as above). The server chooses the concrete type.
- **Distinct collections per type** when they're truly different resources:
  - `POST /gears` vs `POST /springs`
- **Media types / profiles** to drive variation:
  - `Content-Type: application/vnd.example.gear+json`
  - Lets the server pick the concrete constructor by representation type.

All three keep verbs out of the URI and let the uniform interface do the work.

### 3) Hypermedia-guided creation (HATEOAS)

Advertise factory "affordances" in responses so clients discover how to create:

```json
{
  "_links": {
    "self": { "href": "/widgets" },
    "create-gear": {
      "href": "/widgets",
      "method": "POST",
      "schema": "/schemas/gear.json"
    },
    "create-spring": {
      "href": "/widgets",
      "method": "POST",
      "schema": "/schemas/spring.json"
    }
  }
}
```

The client doesn't guess parameters; it follows links and (optionally) dereferences schemas or forms.
### 4) Long-running "factory" jobs (asynchronous build/mint)

If creation is heavy (compile, render, mint), model a job resource:

```http
POST /renders
{ "sceneId": "abc", "quality": "ultra" }

HTTP/1.1 202 Accepted
Location: /render-jobs/7de…
```

Then the client GETs `/render-jobs/7de…` until it exposes a link to the finished artifact:

```json
{
  "status": "done",
  "_links": {
    "result": { "href": "/renders/4b2…" }
  }
}
```

This is a RESTful "factory queue" without RPC verbs.

### 5) Idempotent factory requests (avoid dupes)

Creation via POST isn't inherently idempotent. If you need "exactly-once" semantics, accept an Idempotency-Key header:

```http
Idempotency-Key: 2c8a-…-9fd
```

The server stores the first result and replays it for retries with the same key—still pure REST over HTTP.

### 6) State transitions instead of action endpoints

Avoid `/widgets/create` or `/widgets/compose` as RPC. Creation is a state transition on the collection (it gains a member) performed via POST. Any additional "assembly" steps are further PATCH/PUT state transitions on the created resource or related sub-resources—never verbs-in-URIs.

### 7) Validation, negotiation, and evolution

- Use 415 Unsupported Media Type / 422 Unprocessable Entity for bad inputs.
- Use content negotiation to support alternative creation formats.
- Evolve by adding new link relations (e.g., create-composite-widget) and new profiles/media types—clients discover them via hypermedia.

### Bottom Line

A factory is not a special endpoint with a verb; it's behavior behind a resource-oriented API. Model it as:

- POST to a collection (the "factory floor"),
- optional async job resources for long builds,
- hypermedia to advertise creation options,
- and representations (types/profiles) to pick concrete products.

That's a faithful realization of the factory idea within REST's principles.
2 replies


Question
Most workflow offerings are process driven.
Web presents itself as a collection of resources rather than processes.
How do we model a workflow as a collection of resources that are created, modified, viewed and deleted.
6:31
Answer
This question goes right to the heart of a deep design tension between process-oriented and resource-oriented worldviews.
Let’s unpack it carefully and then connect it to how you can model a workflow as a resource system, REST-style.
:jigsaw: The Tension
ViewpointCore ideaExample metaphorProcess-driven (workflow)The world is a sequence of actions — “Do X, then Y, then Z.”Assembly lineResource-oriented (web/REST)The world is a set of evolving entities (resources) — each with state that changes over time.Living document
:compass: The Key Shift
Instead of modeling the workflow as:
“A set of steps or transitions”
You model it as:
“A collection of resources, each representing a current state or artifact of progress.”
In this worldview, actions are implicit in state transitions of resources.
:gear: Example: Document Approval Workflow
Traditional process mindset:

Draft → Review → Approved → Published
Resource-oriented mindset:

Resource: /documents/{id}
Each state (draft, review, etc.) is a property of the resource, not a separate process step.





# Process Oriented vs Resource Oriented workflows
1. The key mindset shift
A workflow isn’t a script of actions; it’s a system of stateful resources that evolve.
 Each resource embodies its state, and changes to that state happen through standard HTTP semantics — POST, PUT, PATCH, DELETE.
So instead of “doing” something like approve, you express the approval as the creation or update of a related resource that alters the document’s lifecycle.
2. Example: Document approval workflow
Let’s take the same logical workflow:
Draft → In Review → Approved → Published
and reimagine it as a set of resources.
Document resource
GET /documents/42
returns:
{
  "id": "42",
  "title": "Annual Report",
  "status": "in_review",
  "links": {
    "self": "/documents/42",
    "reviews": "/documents/42/reviews"
  }
}
The document itself knows what state it’s in.
 No verbs — just representations of where it is in its lifecycle.
3. Representing actions as subordinate resources
Instead of calling /documents/42/approve, the client creates or modifies a resource that expresses intent.
To approve the document, the reviewer creates a new approval resource under /reviews:
POST /documents/42/reviews
Content-Type: application/json
{
  "reviewer": "bob",
  "decision": "approved",
  "comments": "Looks good."
}
The server, by handling this new review resource, updates the document’s state:
GET /documents/42
{
  "id": "42",
  "status": "approved",
  "links": {
    "self": "/documents/42",
    "reviews": "/documents/42/reviews"
  }
}
No verbs in URLs. The “action” emerges from the creation of a resource that carries intent.
4. Transitions as state changes, not operations
Every state transition corresponds to a change in a resource representation:
Submitting moves a document from "draft" to "in_review"
 → achieved by PATCH /documents/42 { "status": "in_review" }
Approving adds a review resource with "decision": "approved"
Publishing might be represented by PUT /publications/42 that links to the approved document
Each of these reflects HTTP’s semantics:
GET → Retrieve current state
POST → Create new sub-resource or event
PUT / PATCH → Modify representation
DELETE → Remove or deactivate a resource
5. The resource graph
The system becomes a graph of interrelated nouns:
/documents
  /{id}
    /reviews
    /attachments
    /history
/publications
  /{id}
Each node represents something that can exist, be created, updated, deleted, or linked.
 The “workflow” is implicit in allowed transitions between resource states, not in procedural steps.
6. Conceptual model (in JSON)
{
  "resources": [
    {
      "uri": "/documents/42",
      "type": "document",
      "state": "in_review",
      "related": {
        "reviews": "/documents/42/reviews",
        "history": "/documents/42/history"
      },
      "transitions": {
        "on_review_added": { "new_state": "approved" },
        "on_patch_status_in_review": { "new_state": "in_review" }
      }
    }
  ]
}
7. The result
The workflow now lives inside the resource graph, not inside a process engine.
 Each change to a resource is a state transition.
 Each transition is expressed using standard HTTP semantics.
 Each state is discoverable by following links, not by invoking verbs.
This is the essence of a resource-oriented workflow — a RESTful expression of lifecycle and progression through representation, not procedure.


Sridhar Krishnamurthy
  6:52 PM
Saar, look at this !
Let’s take something very tangible and familiar — a leave approval workflow — and show clearly how to transform it from a process-oriented design into a resource-oriented one.
 We’ll go step by step, ending with a simple model of how to represent it.
1. The typical process-oriented view
In a traditional workflow engine, you would describe something like this:
Employee submits → Manager reviews → HR finalizes → Done
In this model:
The process drives everything.
Each step triggers the next.
“Actions” (submit, approve, reject) are verbs — procedural.
2. The resource-oriented mindset
The web is not verb-based.
 It exposes resources (nouns) that can be created, read, modified, or deleted — and whose representations change to reflect workflow state.
So instead of doing actions, we model stateful resources that evolve.
The central noun here is a Leave Request.
3. The domain resources
   Concept Resource URI Description     Leave request /leave-requests/{id} The main object being created and updated   Review / approval /leave-requests/{id}/approvals Sub-resources that represent decisions   User /users/{id} People who interact with requests
4. The lifecycle as resource states
A leave request moves through states:
draft → submitted → approved/rejected → closed
We represent these as resource states, not steps.
Example: Creating a draft
POST /leave-requests
Content-Type: application/json
{
  "employee": "/users/alice",
  "from": "2025-11-12",
  "to": "2025-11-15",
  "reason": "Family event",
  "status": "draft"
}
Response:
201 Created
Location: /leave-requests/42
Submitting the draft
Instead of calling /submit, the employee updates the resource to reflect that it’s been submitted.
PATCH /leave-requests/42
Content-Type: application/json
{
  "status": "submitted"
}
The server updates the record and may create an internal review resource.
Manager approves the request
The manager creates a review resource that expresses approval intent.
POST /leave-requests/42/approvals
Content-Type: application/json
{
  "reviewer": "/users/bob",
  "decision": "approved",
  "comment": "All good"
}
The server now sets:
PATCH /leave-requests/42
{
  "status": "approved"
}
HR closes it
When HR finalizes the process:
PATCH /leave-requests/42
{
  "status": "closed"
}
5. The workflow model as a resource graph
We can represent this system structurally as a graph of resources and transitions:
{
  "resources": [
    {
      "type": "leave-request",
      "uri": "/leave-requests/{id}",
      "states": ["draft", "submitted", "approved", "rejected", "closed"],
      "transitions": [
        { "from": "draft", "to": "submitted", "method": "PATCH", "field": "status" },
        { "from": "submitted", "to": "approved", "trigger": "approval-created" },
        { "from": "submitted", "to": "rejected", "trigger": "approval-created" },
        { "from": "approved", "to": "closed", "method": "PATCH", "field": "status" }
      ],
      "relations": {
        "approvals": "/leave-requests/{id}/approvals",
        "employee": "/users/{id}"
      }
    }
  ]
}
6. What this achieves
No verbs in URIs.
 The API is fully RESTful — only nouns are exposed.
State transitions are expressed as representation changes.
 HTTP PATCH or POST calls mutate resource state.
The workflow logic emerges from state relationships.
 Clients and servers can reason about progression by following links, not hardcoded steps.
Extensible and observable.
 Other services can watch for resource state changes to trigger new behavior.
7. Summary
Process-oriented view:
A sequence of steps that run in order.
Resource-oriented view:
A collection of resources whose states evolve through standard HTTP semantics.
You’ve turned a workflow into a living resource graph.
 The same pattern can model documents, claims, requests, orders, or any process where “doing” can be expressed as “becoming.”


Sridhar Krishnamurthy
  7:22 PM
Here’s a careful, REST-principled, HATEOAS-oriented version of the leave-request example.
 I’ll use a simple HAL-like style with _links and an _actions block to expose affordances (what a client can do next) without putting verbs in URIs. All state changes use POST (to create subordinate resources) or PATCH (to partially update a representation). No PUT is used unless doing a full replacement (which we won’t do here).
Conventions:
Times are illustrative ISO-8601.
Server assigns IDs.
_actions advertise how to evolve state (method, target, and minimal shape), still pointing only to nouns.
Related resources get their own snapshots whenever they change.
0) Create the leave request (POST → new resource)
Request
POST /leave-requests
Content-Type: application/json
{
  "employee": "/users/alice",
  "from": "2025-11-12",
  "to": "2025-11-15",
  "reason": "Family event",
  "status": "draft"
}
Snapshot — /leave-requests/42
{
  "id": "42",
  "type": "leave-request",
  "status": "draft",
  "from": "2025-11-12",
  "to": "2025-11-15",
  "reason": "Family event",
  "employee": "/users/alice",
  "version": 1,
  "created_at": "2025-10-07T08:00:01Z",
  "updated_at": "2025-10-07T08:00:01Z",
  "_links": {
    "self": { "href": "/leave-requests/42" },
    "collection": { "href": "/leave-requests" },
    "approvals": { "href": "/leave-requests/42/approvals" },
    "history": { "href": "/leave-requests/42/history" },
    "employee": { "href": "/users/alice" }
  },
  "_actions": {
    "submit": {
      "method": "PATCH",
      "href": "/leave-requests/42",
      "contentType": "application/json",
      "schema": { "properties": { "status": { "enum": ["submitted"] } }, "required": ["status"] }
    },
    "edit-dates": {
      "method": "PATCH",
      "href": "/leave-requests/42",
      "contentType": "application/json",
      "schema": { "properties": { "from": { "type": "string" }, "to": { "type": "string" }, "reason": { "type": "string" } } }
    }
  }
}
**Snapshot — /leave-requests/42/approvals (collection)**
```json
{
  "type": "collection",
  "items": [],
  "count": 0,
  "updated_at": "2025-10-07T08:00:01Z",
  "_links": {
    "self": { "href": "/leave-requests/42/approvals" },
    "parent": { "href": "/leave-requests/42" }
  }
}
```
**Snapshot — /leave-requests/42/history**
```json
{
  "type": "history",
  "events": [
    {
      "id": "h1",
      "at": "2025-10-07T08:00:01Z",
      "kind": "created",
      "by": "/users/alice",
      "state": "draft"
    }
  ],
  "_links": {
    "self": { "href": "/leave-requests/42/history" },
    "parent": { "href": "/leave-requests/42" }
  }
}
```
1) Submit the draft (PATCH → partial update)
**Request**
```http
PATCH /leave-requests/42
Content-Type: application/json
```

```json
{ "status": "submitted" }
```
**Snapshot — /leave-requests/42**
```json
{
  "id": "42",
  "type": "leave-request",
  "status": "submitted",
  "from": "2025-11-12",
  "to": "2025-11-15",
  "reason": "Family event",
  "employee": "/users/alice",
  "version": 2,
  "created_at": "2025-10-07T08:00:01Z",
  "updated_at": "2025-10-07T08:05:10Z",
  "_links": {
    "self": { "href": "/leave-requests/42" },
    "collection": { "href": "/leave-requests" },
    "approvals": { "href": "/leave-requests/42/approvals" },
    "history": { "href": "/leave-requests/42/history" },
    "employee": { "href": "/users/alice" }
  },
  "_actions": {
    "edit-dates": {
      "method": "PATCH",
      "href": "/leave-requests/42",
      "contentType": "application/json",
      "schema": { "properties": { "from": { "type": "string" }, "to": { "type": "string" }, "reason": { "type": "string" } } }
    },
    "add-approval": {
      "method": "POST",
      "href": "/leave-requests/42/approvals",
      "contentType": "application/json",
      "schema": {
        "properties": {
          "reviewer": { "type": "string" },
          "decision": { "enum": ["approved", "rejected"] },
          "comment": { "type": "string" }
        },
        "required": ["reviewer", "decision"]
      }
    }
  }
}
```
Snapshot — /leave-requests/42/history
{
  "type": "history",
  "events": [
    { "id": "h1", "at": "2025-10-07T08:00:01Z", "kind": "created", "by": "/users/alice", "state": "draft" },
    { "id": "h2", "at": "2025-10-07T08:05:10Z", "kind": "state-change", "by": "/users/alice", "from": "draft", "to": "submitted" }
  ],
  "_links": {
    "self": { "href": "/leave-requests/42/history" },
    "parent": { "href": "/leave-requests/42" }
  }
}
2) Correct dates (PATCH → partial update)
Request
PATCH /leave-requests/42
Content-Type: application/json
{
  "from": "2025-11-13",
  "to": "2025-11-16",
  "reason": "Family event (updated)"
}
Snapshot — /leave-requests/42
{
  "id": "42",
  "type": "leave-request",
  "status": "submitted",
  "from": "2025-11-13",
  "to": "2025-11-16",
  "reason": "Family event (updated)",
  "employee": "/users/alice",
  "version": 3,
  "created_at": "2025-10-07T08:00:01Z",
  "updated_at": "2025-10-07T08:07:30Z",
  "_links": {
    "self": { "href": "/leave-requests/42" },
    "collection": { "href": "/leave-requests" },
    "approvals": { "href": "/leave-requests/42/approvals" },
    "history": { "href": "/leave-requests/42/history" },
    "employee": { "href": "/users/alice" }
  },
  "_actions": {
    "add-approval": {
      "method": "POST",
      "href": "/leave-requests/42/approvals",
      "contentType": "application/json",
      "schema": {
        "properties": {
          "reviewer": { "type": "string" },
          "decision": { "enum": ["approved", "rejected"] },
          "comment": { "type": "string" }
        },
        "required": ["reviewer", "decision"]
      }
    }
  }
}
**Snapshot — /leave-requests/42/history**
```json
{
  "type": "history",
  "events": [
    { "id": "h1", "at": "2025-10-07T08:00:01Z", "kind": "created", "by": "/users/alice", "state": "draft" },
    { "id": "h2", "at": "2025-10-07T08:05:10Z", "kind": "state-change", "by": "/users/alice", "from": "draft", "to": "submitted" },
    { "id": "h3", "at": "2025-10-07T08:07:30Z", "kind": "fields-updated", "by": "/users/alice", "patch": ["from", "to", "reason"] }
  ],
  "_links": {
    "self": { "href": "/leave-requests/42/history" },
    "parent": { "href": "/leave-requests/42" }
  }
}
```
3) Manager records a decision (POST → create subordinate resource)
**Request**
```http
POST /leave-requests/42/approvals
Content-Type: application/json
```

```json
{
  "reviewer": "/users/bob",
  "decision": "approved",
  "comment": "All good"
}
```
**Snapshot — /leave-requests/42/approvals/a1 (new)**
```json
{
  "id": "a1",
  "type": "approval",
  "decision": "approved",
  "comment": "All good",
  "reviewer": "/users/bob",
  "request": "/leave-requests/42",
  "created_at": "2025-10-07T08:10:05Z",
  "_links": {
    "self": { "href": "/leave-requests/42/approvals/a1" },
    "collection": { "href": "/leave-requests/42/approvals" },
    "reviewer": { "href": "/users/bob" },
    "request": { "href": "/leave-requests/42" }
  }
}
```
**Snapshot — /leave-requests/42/approvals (collection updated)**
```json
{
  "type": "collection",
  "items": [ "/leave-requests/42/approvals/a1" ],
  "count": 1,
  "updated_at": "2025-10-07T08:10:05Z",
  "_links": {
    "self": { "href": "/leave-requests/42/approvals" },
    "parent": { "href": "/leave-requests/42" }
  }
}
```
**Snapshot — /leave-requests/42 (state advances due to domain rule)**
```json
{
  "id": "42",
  "type": "leave-request",
  "status": "approved",
  "from": "2025-11-13",
  "to": "2025-11-16",
  "reason": "Family event (updated)",
  "employee": "/users/alice",
  "version": 4,
  "created_at": "2025-10-07T08:00:01Z",
  "updated_at": "2025-10-07T08:10:05Z",
  "_links": {
    "self": { "href": "/leave-requests/42" },
    "collection": { "href": "/leave-requests" },
    "approvals": { "href": "/leave-requests/42/approvals" },
    "latest-approval": { "href": "/leave-requests/42/approvals/a1" },
    "history": { "href": "/leave-requests/42/history" },
    "employee": { "href": "/users/alice" }
  },
  "_actions": {
    "close": {
      "method": "PATCH",
      "href": "/leave-requests/42",
      "contentType": "application/json",
      "schema": { "properties": { "status": { "enum": ["closed"] } }, "required": ["status"] }
    }
  }
}
```
**Snapshot — /leave-requests/42/history**
```json
{
  "type": "history",
  "events": [
    { "id": "h1", "at": "2025-10-07T08:00:01Z", "kind": "created", "by": "/users/alice", "state": "draft" },
    { "id": "h2", "at": "2025-10-07T08:05:10Z", "kind": "state-change", "by": "/users/alice", "from": "draft", "to": "submitted" },
    { "id": "h3", "at": "2025-10-07T08:07:30Z", "kind": "fields-updated", "by": "/users/alice", "patch": ["from", "to", "reason"] },
    { "id": "h4", "at": "2025-10-07T08:10:05Z", "kind": "approval-recorded", "by": "/users/bob", "decision": "approved" },
    { "id": "h5", "at": "2025-10-07T08:10:05Z", "kind": "state-change", "by": "/system", "from": "submitted", "to": "approved" }
  ],
  "_links": {
    "self": { "href": "/leave-requests/42/history" },
    "parent": { "href": "/leave-requests/42" }
  }
}
```
4) HR closes the request (PATCH → partial update)
**Request**
```http
PATCH /leave-requests/42
Content-Type: application/json
```

```json
{ "status": "closed" }
```
**Snapshot — /leave-requests/42**
```json
{
  "id": "42",
  "type": "leave-request",
  "status": "closed",
  "from": "2025-11-13",
  "to": "2025-11-16",
  "reason": "Family event (updated)",
  "employee": "/users/alice",
  "version": 5,
  "created_at": "2025-10-07T08:00:01Z",
  "updated_at": "2025-10-07T08:12:40Z",
  "_links": {
    "self": { "href": "/leave-requests/42" },
    "collection": { "href": "/leave-requests" },
    "approvals": { "href": "/leave-requests/42/approvals" },
    "latest-approval": { "href": "/leave-requests/42/approvals/a1" },
    "history": { "href": "/leave-requests/42/history" },
    "employee": { "href": "/users/alice" }
  },
  "_actions": { }
}
```
**Snapshot — /leave-requests/42/history**
```json
{
  "type": "history",
  "events": [
    { "id": "h1", "at": "2025-10-07T08:00:01Z", "kind": "created", "by": "/users/alice", "state": "draft" },
    { "id": "h2", "at": "2025-10-07T08:05:10Z", "kind": "state-change", "by": "/users/alice", "from": "draft", "to": "submitted" },
    { "id": "h3", "at": "2025-10-07T08:07:30Z", "kind": "fields-updated", "by": "/users/alice", "patch": ["from", "to", "reason"] },
    { "id": "h4", "at": "2025-10-07T08:10:05Z", "kind": "approval-recorded", "by": "/users/bob", "decision": "approved" },
    { "id": "h5", "at": "2025-10-07T08:10:05Z", "kind": "state-change", "by": "/system", "from": "submitted", "to": "approved" },
    { "id": "h6", "at": "2025-10-07T08:12:40Z", "kind": "state-change", "by": "/users/hr", "from": "approved", "to": "closed" }
  ],
  "_links": {
    "self": { "href": "/leave-requests/42/history" },
    "parent": { "href": "/leave-requests/42" }
  }
}
```
Why this is REST-friendly & HATEOAS-oriented
No verbs in URIs. All endpoints are nouns (resources and collections).
Hypermedia drives the client. _links shows where to navigate; _actions advertises how to evolve state (method + target + minimal schema).
HTTP semantics map to intent. POST creates a subordinate resource (an approval), which induces a state change on the parent per domain rules; PATCH updates parts of the representation (status, dates).
Related resources are first-class. Approvals and history are their own resources with their own snapshots and links.
If you want, I can also provide a compact JSON Schema for each resource type (leave-request, approvals collection, approval item, history) to accompany this example.




# SIREN


## Siren Resources

- [Siren Specification - GitHub](https://github.com/kevinswiber/siren)
- [A Good Example of Hypermedia API Using the Siren Specification](https://apievangelist.com/2015/05/11/a-good-example-of-hypermedia-api-using-the-siren-specification/)
- [Spring HATEOAS Siren Documentation](https://spring-hateoas-siren.ingogriebsch.de/current/index.html)
- [Hypertext Application Language - Wikipedia](https://en.wikipedia.org/wiki/Hypertext_Application_Language)
- [RFC 8288 - Web Linking](https://www.rfc-editor.org/rfc/rfc8288.txt)
- [IANA Link Relations Registry](https://www.iana.org/assignments/link-relations/link-relations.xhtml)
- [RFC 6570 - URI Template](https://www.rfc-editor.org/rfc/rfc6570.txt)
- [RFC 6906 - The 'profile' Link Relation Type](https://www.rfc-editor.org/rfc/rfc6906.txt)
- [HAL Draft Specification](https://datatracker.ietf.org/doc/html/draft-kelly-json-hal-11)
- [HTML Input Element Specification](https://html.spec.whatwg.org/#the-input-element)
(edited)
9 replies


Sridhar Krishnamurthy
  Oct 7th at 8:14 PM
Siren is a hypermedia specification for representing entities in a Web API, designed to facilitate communication of both data and actionable operations to clients. Unlike simpler hypermedia formats that primarily focus on links, Siren provides a richer structure that enables clients to understand not only how to navigate the API but also what actions they can perform.
Key components of the Siren specification include:
Entities: The core building blocks of a Siren response. An entity represents a resource and encapsulates its properties, actions, links, and sub-entities.
Properties: The data associated with an entity, defining its current state. These are typically key-value pairs.
Actions: Operations that clients can perform on an entity, leading to state transitions. Actions include details like the HTTP method (e.g., PUT, POST), target URI, and encoding type.
Links: Navigational options to related resources, similar to traditional web links. Links include a rel (relation) attribute to define their relationship to the entity and an href (hypertext reference) for the target URI.
Sub-entities: Nested entities that represent related resources within a hierarchical structure, providing a way to embed associated data directly within a parent entity.
Classes: Semantic labels (arrays of strings) that provide additional context for entities and actions, helping clients interpret their purpose and role within the API.
Siren is particularly well-suited for APIs that involve complex workflows and dynamic interactions, as it allows for the explicit representation of available actions and guides clients through multi-step processes. By combining data with actionable operations, Siren enables clients to adapt their behavior based on the current state of the entities, fostering a more interactive and self-descriptive API experience.


Sridhar Krishnamurthy
  Oct 7th at 8:21 PM
The connection that ties Siren and HATEOAS together beautifully.
Let’s go carefully and clearly, in plain text, so you can share or reuse this easily.
1. What HATEOAS really means
HATEOAS stands for Hypermedia As The Engine Of Application State.
 It’s one of the core constraints of REST, defined by Roy Fielding.
The idea is simple but powerful:
A REST client should be able to discover what it can do next by looking at the current resource representation, not by relying on out-of-band rules or hardcoded endpoint knowledge.
That means:
The server controls the workflow.
The client follows links and actions advertised in the current representation.
Each representation includes affordances (possible transitions) — not just data.
So instead of saying
 “After /leave-requests/42 is approved, call /close,”
 the representation itself tells the client what actions are available.
2. Where Siren fits in
Siren is a hypermedia format designed specifically to make HATEOAS practical and expressive in JSON.
It defines a standard vocabulary for embedding:
entities (resources and sub-resources)
links (navigation)
actions (state transitions and operations)
So with Siren, each resource can say:
“Here’s my current state” (via properties)
“Here’s what I’m related to” (via entities and links)
“Here’s what you can do next” (via actions)
That’s HATEOAS in action.
3. Siren enables HATEOAS through structure
When a client fetches a Siren document:
It looks at properties → gets the resource data.
It inspects links and entities → learns about related resources.
It inspects actions → knows what transitions are possible and how to perform them.
The client doesn’t need to know the workflow in advance.
 If the workflow changes, the server just emits a different set of actions.
 Clients remain generic — they only need to know Siren rules, not domain rules.
4. Why Siren is well-suited to HATEOAS
Siren representations are self-contained, self-describing, and stateful.
 Each one carries both:
The current application state (properties, class)
The available transitions (actions)
For example, a leave request in the “approved” state might only show one action:
```json
"actions": [
  {
    "name": "close",
    "method": "PATCH",
    "href": "/leave-requests/42",
    "type": "application/json",
    "fields": [
      { "name": "status", "type": "hidden", "value": "closed" }
    ]
  }
]
```
That’s the HATEOAS principle:
the representation itself tells the client what’s next.
5. Benefits of combining Siren and HATEOAS
Discoverability
 Clients discover new workflows dynamically through actions and links.
Decoupling
 Clients and servers evolve independently. The server defines state transitions through representations; clients follow the Siren grammar.
Generic clients
 A generic Siren client can render buttons, forms, and navigation automatically, without knowing business logic.
Version tolerance
 New actions or fields can appear without breaking clients — they can ignore what they don’t understand.
True REST uniformity
 Every transition uses standard HTTP semantics (GET, POST, PATCH, DELETE) rather than custom verbs in URLs.
6. How the pieces align
Think of it like layers of meaning:
HTTP → defines methods, status codes, caching.
 Siren → defines hypermedia structure (links, entities, actions).
 Profile → defines domain semantics (what a “leave request” means).
 Representation → combines all three into a discoverable workflow state.
Together they embody HATEOAS — the application state flows from representation to representation through hypermedia controls.
7. The core idea to remember
Siren is not HATEOAS itself — it is an enabler of HATEOAS.
 HATEOAS is the REST principle.
 Siren is one elegant way to implement that principle in JSON.
When you use Siren:
Each resource describes its current state and valid transitions.
The client never hardcodes the next step.
The web of resources itself drives the application.
That is exactly what “Hypermedia as the Engine of Application State” was meant to achieve. (edited) 


Sridhar Krishnamurthy
  Oct 8th at 12:09 PM
Great — this is one of the most important comparisons to understand when you’re designing RESTful hypermedia APIs.
Let’s look at HAL and Siren side by side, in clear prose (no tables).
1. Their shared goal
Both HAL (Hypertext Application Language) and Siren were created to make REST APIs discoverable and self-describing, so that clients could follow links instead of hardcoding routes or verbs.
In other words, both exist to enable HATEOAS — but they take very different approaches.
2. HAL — minimal, link-focused hypermedia
HAL is the older, simpler specification.
 Its goal is to make linking between JSON resources uniform.
A HAL document adds two reserved properties:
_links — for hyperlinks (e.g., self, related, next, previous)
_embedded — for inlining related resources
Everything else is plain resource data.
HAL focuses on relationships between resources, not on interactions with them.
It says: “Here’s this resource and here’s what it’s connected to.”
 But it does not say: “Here’s what you can do next.”
So HAL is great for navigating and embedding, but not for expressing actions or workflows.
Example of HAL:
```json
{
  "id": 42,
  "title": "Annual Report",
  "_links": {
    "self": { "href": "/documents/42" },
    "collection": { "href": "/documents" },
    "reviews": { "href": "/documents/42/reviews" }
  },
  "_embedded": {
    "reviews": [
      { "id": "r1", "reviewer": "bob", "decision": "approved" }
    ]
  }
}
```
This helps a client browse the data graph, but it still needs out-of-band rules to know what can be done next.
3. Siren — full hypermedia with actions and state
Siren builds on the same idea but goes much further.
It treats each resource as an entity that can contain:
properties — the current state
entities — sub-resources or related resources
links — navigation
actions — possible operations, including method, href, fields
class — semantic hints about type
title — human-readable name
Where HAL shows links, Siren shows affordances — what actions can be performed and what fields are needed.
Siren therefore models both data and behavior in one structure.
Example of Siren:
```json
{
  "class": ["document"],
  "properties": {
    "id": 42,
    "title": "Annual Report",
    "status": "in_review"
  },
  "actions": [
    {
      "name": "approve",
      "method": "PATCH",
      "href": "/documents/42",
      "type": "application/json",
      "fields": [{ "name": "status", "type": "hidden", "value": "approved" }]
    }
  ],
  "links": [
    { "rel": ["self"], "href": "/documents/42" },
    { "rel": ["collection"], "href": "/documents" }
  ]
}
```
Now a client can see both:
what the document is (its state)
what can be done (the “approve” action)
4. Their design philosophies
HAL aims to be minimal and easy to parse — just enough structure to link resources consistently.
 Siren aims to be expressive — a full grammar for building hypermedia-driven applications.
HAL is ideal for APIs that are mostly data retrieval or navigational.
 Siren is ideal for APIs that represent workflows, tasks, and state transitions.
You can think of it this way:
 HAL is like a map — it tells you where things are.
 Siren is like a user interface — it tells you what you can do next.
5. REST conformance
Both HAL and Siren conform to REST and HATEOAS principles.
 They differ only in expressiveness.
HAL supports HATEOAS at the link level.
Siren supports HATEOAS at the interaction level — including actions and transitions.
A HAL client needs extra rules to know how to change state; a Siren client can infer that directly from the representation.
6. Coexistence and evolution
Many teams start with HAL because it’s simple to layer over existing JSON APIs.
 As APIs evolve toward workflows or UI-driven behaviors, they move to Siren or similar formats like Siren, Hydra, or Collection+JSON, which include actions.
In practice, you can mix them — use HAL for simple resources and Siren for interactive ones.
7. Summary comment
HAL gives you a way to connect resources.
 Siren gives you a way to drive them.
Both are RESTful.
 HAL is light and link-centric; Siren is rich and action-centric.
 If your system needs the client to “walk the graph,” use HAL.
 If it needs the client to “walk the graph and perform operations,” use Siren.


Sridhar Krishnamurthy
  Oct 8th at 12:58 PM
Here’s a clean, readable summary of RFC 8288 (titled “Web Linking”), the IETF standard that defines how links work in HTTP and other web contexts.
1. Purpose
RFC 8288 defines a standard model for links on the Web — what a “link” is, how to describe it, and how to transmit it (for example, in HTTP headers or within representations).
It gives us a consistent, machine-readable way to say:
“This resource is related to that other resource — and here’s what that relationship means.”
So it generalizes the concept of HTML <a href> links to apply everywhere on the Web, including APIs, metadata, and non-HTML resources.
2. The core concept — a Web link
A link has two parts:
a target IRI/URI — the thing being linked to
a relation type (rel) — a label describing how the two are related
Optional attributes can add context (media type, language, title, etc.).
Example of a simple link in HTTP header form:
Link: <https://example.com/help>; rel="help"
This means:
The current resource has a relationship “help” pointing to https://example.com/help.
3. Syntax and transmission
RFC 8288 defines how links are expressed:
In HTTP headers:
Link: <target-URI>; rel="relation"; type="type/subtype"; hreflang="en"
In documents:
 Links can also appear inside representations (HTML, JSON, XML, etc.) — the same model applies regardless of format.
A resource can include multiple Link headers, and each can include multiple link-values separated by commas.
Example with multiple links:
Link: <https://api.example.com/users?page=2>; rel="next",
      <https://api.example.com/users?page=5>; rel="last"
4. Relation types (rel)
The rel attribute identifies the semantic meaning of the link — what the relationship is between the current resource and the target.
Relation types can be:
registered in the IANA Link Relation Registry (e.g., self, next, prev, about, alternate, help, license, etc.)
or extension relation types, identified by an absolute URI.
Examples:
Link: <https://example.com/about>; rel="about"
Link: <https://schema.org/Person>; rel="profile"
5. Parameters and attributes
You can add attributes to refine meaning or provide metadata:
type — expected media type of the target
hreflang — language of the target
title — human-readable label
anchor — defines which resource the link applies to (defaults to the current one)
Example:
Link: <https://example.com/report.pdf>; rel="alternate"; type="application/pdf"; hreflang="en"
6. The Link header field
This is the key addition to HTTP.
 It allows resources to advertise links without modifying the body.
Clients or intermediaries can look at headers alone to discover relationships.
The general form:
Link: <target-URI-reference>; param1=value1; param2=value2
Multiple values may appear in one header line, separated by commas.
7. The rel="profile" usage (important for APIs)
The spec defines a relation type called profile.
 This is widely used by hypermedia formats (like HAL, Siren, JSON-LD, etc.) to point to a URI that defines additional semantics about the representation.
Example:
Content-Type: application/vnd.siren+json; profile="https://example.com/profiles/leave-request"
Link: <https://example.com/profiles/leave-request>; rel="profile"
This tells clients how to interpret the resource semantics — not just the syntax.
8. Key design principles
Links are typed, not verbs.
 The rel attribute defines the relationship, not an action.
Links are directional.
 They describe a relationship from a context resource to a target resource.
Links are independent of representation.
 A link can be expressed in HTTP headers or embedded in content — same semantics.
Links are reusable across media types.
 The model is universal — HTML, JSON, Atom, HAL, Siren all use the same foundation.
9. Registry and extensibility
RFC 8288 establishes the IANA Link Relation Type Registry.
 Developers can register new rel values to standardize link meanings.
For example:
self → identifies the current resource
next / prev → pagination
related → general association
profile → link to a profile describing representation semantics
collection → a resource that groups related items
This makes links interoperable and machine-readable across systems.
10. In essence
RFC 8288 defines the universal linking vocabulary of the Web.
It standardizes:
What a “link” means
How to serialize it in HTTP
How to describe relationships between resources
How to register and use relation types safely
Every modern hypermedia system — HAL, Siren, Hydra, JSON:API, Atom, and many RESTful APIs — builds on the foundation this RFC provides.
11. The takeaway idea
A Web link is not just an href.
 It’s a structured statement:
“This resource (context) has a relationship (rel) to that other resource (target), optionally with metadata about how to use it.”
RFC 8288 formalizes that model so any client, not just browsers, can reason about resource relationships in a consistent, RESTful way.


Sridhar Krishnamurthy
  Oct 8th at 1:46 PM
Excellent — this is a subtle but vital connection, and understanding it makes HAL’s design click into place.
Let’s walk through the relationship between RFC 8288 (“Web Linking”) and HAL’s _links structure clearly and in plain text.
1. What RFC 8288 established
RFC 8288 defined what a Web link is in general terms.
 A link has a
context (the resource that owns the link),
relation type (the rel value — like self, next, collection),
target (a URI), and
optional attributes (type, hreflang, title, anchor, etc.).
It also standardized the HTTP Link header so servers could send typed links independently of the body, for example:
Link: <https://api.example.com/users?page=2>; rel="next"
RFC 8288 thus gave the Web a universal, machine-readable model for typed relationships between resources.
2. HAL applied RFC 8288 inside JSON
HAL’s core idea is simple:
take the link model from RFC 8288 and embed it inside a JSON representation so APIs can express the same semantics in the response body.
So HAL’s _links object is effectively a JSON serialization of the Link header model defined by RFC 8288.
For example:
{
  "_links": {
    "self": { "href": "/orders/123" },
    "next": { "href": "/orders/124" },
    "collection": { "href": "/orders" }
  },
  "id": 123,
  "status": "shipped"
}
Each entry inside _links corresponds to a Link header entry:
Link: </orders/123>; rel="self"
Link: </orders/124>; rel="next"
Link: </orders>; rel="collection"
The structure matches RFC 8288’s model exactly — the rel value becomes the key, and href is the target URI.
3. Why HAL and RFC 8288 align so closely
Both use the same concept of typed relationships (rel values).
Both can use the IANA Link Relation Registry, which RFC 8288 formalized.
Both allow extension attributes such as type, hreflang, and title on each link object.
HAL simply reuses this RFC 8288 model in a JSON-friendly way, making it easy for clients to discover and navigate relationships within the same representation, without having to parse HTTP headers.
4. What HAL adds
HAL adds two main things beyond what RFC 8288 provides:
A consistent JSON vocabulary (_links and _embedded) so you can express link relationships and embedded resources in one place.
A lightweight convention for representing graph navigation within APIs.
But semantically, it remains faithful to the link model of RFC 8288 — every _links entry is a structured, typed relationship as the RFC defines it.
5. The deeper takeaway
RFC 8288 is the theory and standard of Web linking.
 HAL is one practical implementation of that theory inside JSON.
Put simply:
HAL is RFC 8288 “Web Linking,” embedded inside resource representations instead of HTTP headers.
That’s why HAL’s _links object feels so natural in REST — it’s literally the Web Linking model applied to your API’s JSON responses.


Sridhar Krishnamurthy
  Oct 8th at 2:08 PM
Here’s a clear and straightforward summary of RFC 6570 — URI Template (published by the IETF, authored by Joe Gregorio).
1. Purpose
RFC 6570 defines a standard way to create and expand URI templates — strings that contain variables which can be substituted with actual values to generate valid URIs.
It provides a language for URI construction, so a client can safely build or discover links from templates provided by a server, without hardcoding URI patterns.
Example conceptually:
Template: /users/{userid}/orders{?page,size}
Expanded: /users/alice/orders?page=2&size=10
2. Why it matters
On the Web, resources are often related in predictable ways.
 Instead of forcing clients to know exact URLs, servers can expose templates that describe how URLs are structured.
This supports the HATEOAS principle — the client can follow links and expand templates dynamically rather than relying on out-of-band rules.
3. The basic model
A URI template is a string with variable expressions enclosed in {}.
 When you expand the template, you replace each variable with one or more values.
Simple example:
Template: /repos/{owner}/{repo}
Variables: { "owner": "example", "repo": "demo" }
Result: /repos/example/demo
4. Expression types and operators
The RFC defines several operators that control how values are expanded and encoded:
Level 1 (Simple String Expansion)
 {var} → replaces with the value (URL-encoded).
 Example: /users/{user} → /users/alice
Level 2 (Reserved Expansion)
 {+var} → allows reserved characters (like /, ?, :) to remain unescaped.
 Example: /path/{+file} → /path/images/logo.png
Level 3 (Fragment Expansion)
 {#var} → same as reserved, but prefixed with # for fragments.
 Example: {#section} → #intro
Level 4 (Query Expansion and more)
 Adds more operators for complex structures:
? — starts query string: {?x,y} → ?x=1&y=2
& — adds to existing query: {&x,y} → &x=1&y=2
; — path-style parameters: {;x,y} → ;x=1;y=2
/ — path segment expansion: {/x,y} → /1/2
. — dot-prefixed: {.x,y} → .1.2
You can also use lists and maps to expand multiple values:
Template: /search{?keywords*}
Variables: { "keywords": ["http", "uri", "template"] }
Result: /search?keywords=http&keywords=uri&keywords=template
5. Variable naming and encoding
Variables can be simple names or dotted paths (user.name).
 Values are percent-encoded as needed to ensure valid URIs.
 Reserved expansions (+, #) preserve reserved characters.
6. Levels of template complexity
RFC 6570 defines four levels of template support:
Level 1: Simple string expansion ({var})
Level 2: Adds reserved and fragment expansions ({+var}, {#var})
Level 3: Adds dot, path, and parameter expansions
Level 4: Adds query expansions, list and map handling
Implementations can claim compliance up to a specific level.
7. Integration with REST and hypermedia
URI templates appear in many REST and hypermedia formats:
 HAL, Siren, JSON-LD, Hydra, and others use them in templated links.
For example, HAL might include:
"_links": {
  "search": {
    "href": "/orders{?id,status}",
    "templated": true
  }
}
Clients recognize "templated": true and expand using RFC 6570 rules.
This enables discoverability and flexibility — servers expose structure; clients fill in variables.
8. Summary idea
RFC 6570 — URI Template defines a precise grammar and algorithm for safely turning variable expressions into valid URIs.
It standardizes:
how to express variables within URI patterns
how to expand them deterministically
how to handle reserved characters, query parameters, and composite values
It underpins many modern hypermedia APIs by letting servers describe link templates that clients can expand dynamically, making RESTful APIs self-descriptive and adaptable.


Sridhar Krishnamurthy
  Oct 8th at 2:16 PM
Here’s a clear, structured summary of RFC 6906 — “The ‘profile’ Link Relation Type” (by Mark Nottingham, 2013).
1. Purpose
RFC 6906 defines the profile link relation type, a simple but powerful mechanism for adding extra semantics to a resource representation without changing its media type.
In essence, it answers the question:
“How can I tell clients that this application/json (or application/hal+json, etc.) document follows a particular domain convention or schema?”
It provides a formal, RESTful way to declare what kind of resource a representation is — what rules, vocabulary, or semantics apply to it.
2. The problem it solves
Many APIs use generic media types such as application/json or application/hal+json.
 But these types only tell clients how to parse the data, not what it means.
For example:
Is this JSON describing a User, a Leave Request, or a Product?
What fields and relations should a client expect?
Creating a new custom media type for every resource (application/vnd.example.user+json, etc.) is heavy and difficult to manage.
 RFC 6906 introduces profiles as a lightweight alternative.
3. What a “profile” is
A profile is a resource (identified by a URI) that provides additional semantics — documentation, constraints, or processing rules — for the representation.
It might point to:
a human-readable document (like HTML docs or a wiki)
a machine-readable definition (like JSON Schema, SHACL, ALPS, RDF vocabularies)
or both
The profile does not change the media type; it just adds meaning.
4. How it’s used
You can associate a profile with a representation in two ways:
In the Content-Type header (as a parameter):
Content-Type: application/hal+json; profile="https://example.com/profiles/leave-request"
As a Link header:
Link: <https://example.com/profiles/leave-request>; rel="profile"
Either way, the profile URI identifies the set of semantics the representation adheres to.
5. What profiles do not do
A profile does not change how the media type is parsed or processed.
 The representation is still application/hal+json, application/siren+json, etc.
A profile does not need to be dereferenceable (although it can be).
 It just needs to uniquely identify the semantic definition.
So a client can safely treat unknown profiles as opaque identifiers.
6. Example
If a server returns:
Content-Type: application/vnd.siren+json;profile="https://example.com/profiles/leave-request"
Then the client knows:
The document follows the Siren hypermedia format.
The data specifically represents a Leave Request, defined by the given profile URI.
This allows generic Siren clients to render or validate the document correctly, possibly loading a renderer or schema associated with that profile.
7. Key benefits
Decouples semantics from syntax: You don’t need a new media type for every domain object.
Supports content negotiation: Clients can ask for specific profiles.
Promotes discoverability: Clients can follow the rel="profile" link to learn about the resource.
Compatible with HATEOAS: Profiles fit neatly with hypermedia formats like HAL, Siren, or Hydra.
8. Typical usage in APIs
A RESTful API might define profiles such as:
https://api.example.com/profiles/user
https://api.example.com/profiles/order
https://api.example.com/profiles/invoice
Then return:
Content-Type: application/hal+json;profile="https://api.example.com/profiles/order"
Link: <https://api.example.com/profiles/order>; rel="profile"
Clients or renderers can use those profile URIs to interpret meaning, load validation schemas, or render correct UI components.
9. The takeaway idea
RFC 6906 introduced profiles as lightweight semantic identifiers for Web resources.
It lets APIs and representations:
stay with standard media types (application/json, application/hal+json, application/vnd.siren+json)
yet still describe what kind of resource is being represented
using a single, simple mechanism (profile link relation or parameter)
10. In short
RFC 6906 gives us a standardized, RESTful way to declare “what this resource means”
 — not by inventing new MIME types, but by adding a profile that points to the definition of its semantics.


Sridhar Krishnamurthy
  Oct 8th at 2:45 PM
Excellent — this is a subtle but very important part of how RFC 6906 (Profiles) works, and how it interacts with RFC 8288 (Web Linking).
Let’s break down those two lines and clarify what each one does and why both can appear together.
Line 1
Content-Type: application/hal+json;profile="https://api.example.com/profiles/order"
This line tells the client two things at once:
The format of the representation is application/hal+json — so it follows the HAL structure with _links and _embedded.
The profile identifies additional semantics:
 "https://api.example.com/profiles/order" points to a description of what this resource means — in this case, an Order resource.
So the media type still defines how to parse the document (HAL in JSON),
 while the profile URI defines how to interpret its content (Order-specific semantics, fields, relations, etc.).
The profile doesn’t alter parsing rules; it simply adds meaning on top of the base format.
You can think of this as two layers of information in the same header:
syntax = application/hal+json
semantics = profile="https://api.example.com/profiles/order"
Line 2
Link: <https://api.example.com/profiles/order>; rel="profile"
This is an explicit link from the resource to its profile — using the Link header defined by RFC 8288 (Web Linking) and the profile relation type defined by RFC 6906.
It’s functionally similar to the parameter in the Content-Type, but it provides that link independently of the representation.
That means even if a proxy or cache only inspects HTTP headers (without reading the body), it still knows the resource is associated with that profile.
It also allows multiple profiles if needed — for example:
Link: <https://api.example.com/profiles/order>; rel="profile",
      <https://schema.org/Order>; rel="profile"
Why both are shown together
You can include the profile in either the Content-Type parameter or the Link header.
 They serve the same conceptual purpose but at different scopes:
The profile parameter inside Content-Type attaches semantics to the specific representation being returned (this exact body, right now).
The Link: rel="profile" header attaches semantics to the resource itself, independent of a specific representation format.
Using both together is completely valid and sometimes desirable — it provides maximum clarity to different consumers:
Parsers and renderers use the profile parameter (they already look at Content-Type).
Metadata-aware tools, caches, and proxies can also see the profile link in headers without parsing the body.
The resulting meaning
These two lines together tell the client:
“This representation is HAL JSON data, and it represents an Order resource whose semantics are defined at https://api.example.com/profiles/order.”
That URI could point to:
an HTML document explaining the Order structure for humans,
or a machine-readable schema (JSON Schema, ALPS, SHACL, etc.) for clients.
Summary in one sentence
Content-Type: …;profile="…"  ties a profile to this specific representation’s semantics,
 while
 Link: <…>; rel="profile" exposes that same relationship at the HTTP header level,
 so intermediaries and generic clients can also recognize the profile — both ways express the same association defined by RFC 6906.


Sridhar Krishnamurthy
  Oct 8th at 2:53 PM
Perfect — here’s a clean, easy-to-share explanation of the media-type layering model that connects everything we’ve discussed so far: JSON, HAL, Siren, profiles, and REST semantics.
1. The core idea
Each layer of a representation adds a different kind of meaning.
 You can imagine the stack like this:
HTTP (protocol)
  ↓
Media type (syntax and structure)
  ↓
Profile (domain semantics)
  ↓
Representation instance (data + links + actions)
Each layer builds on the one below it.
2. Step by step
HTTP
 Defines the communication mechanics — methods (GET, POST, PATCH), caching, headers, and status codes.
 It doesn’t say anything about what’s inside the message.
Media type
 Defines how to interpret the representation’s structure.
 Examples:
application/json → raw data only
application/hal+json → data plus _links and _embedded
application/vnd.siren+json → data plus actions, entities, links
This level describes format semantics — how the data is organized, not what it represents.
Profile
 Adds domain-specific meaning on top of the format.
 Example:
Content-Type: application/vnd.siren+json; profile="https://example.com/profiles/leave-request"
Now clients know the data follows the Siren format and that it represents a Leave Request according to the rules defined by that profile.
Profiles can point to:
JSON Schema documents (for validation)
ALPS descriptors (for semantic actions)
HTML docs (for human understanding)
Representation instance
 This is the actual resource state being transferred — for example, the Siren document for /leave-requests/42 that includes links, actions, and properties.
3. How the layers interact
The server declares the format (media type) and semantics (profile) in the Content-Type header.
The client uses those to choose the right parser and renderer.
Hypermedia links (_links, actions, etc.) lead the client to the next state.
Both client and server can evolve independently, because meaning is discoverable and layered, not hardcoded.
4. Putting it all together with example headers
GET /leave-requests/42
Accept: application/vnd.siren+json;profile="https://example.com/profiles/leave-request"
Response:
200 OK
Content-Type: application/vnd.siren+json;profile="https://example.com/profiles/leave-request"
Link: <https://example.com/profiles/leave-request>; rel="profile"
The client now knows:
It’s a Siren document (so it can parse actions, links, entities).
It’s a Leave Request (so it can render or validate domain-specific fields).
It can follow hypermedia links to move through workflow states — true HATEOAS.
5. The conceptual summary
HTTP gives the transport semantics.
The media type gives the syntactic and structural semantics.
The profile gives the domain semantics.
The representation instance delivers the current application state.
Each layer does one job, cleanly separated — the essence of REST’s layered uniform interface.
Would you like me to extend this model to show how URI Templates (RFC 6570) and Web Linking (RFC 8288) fit into this same layered picture? They’re the other two building blocks that complete the hypermedia ecosystem.



# ALPS
Excellent question — and since you’re now working with REST semantics, media types, and profiles, ALPS is the perfect next concept to add to your toolkit.
Let’s go through this carefully, step by step, so you get a complete, shareable understanding.
1. What ALPS stands for
ALPS means Application-Level Profile Semantics.
It’s a specification published by the IETF (drafted by Mike Amundsen and others) that defines a simple, machine-readable way to describe the semantics of a RESTful application — without binding those semantics to any specific format like JSON, XML, or HTML.
So while a JSON Schema describes structure (“this field is a string”),
 an ALPS document describes meaning and intent (“this field represents a person’s name”).
2. What an ALPS document does
An ALPS document is a profile in the RFC 6906 sense —
 it defines what kinds of data, actions, and transitions exist in an application domain.
Each ALPS document contains descriptors, which are the building blocks of your semantics.
 They describe:
data elements (fields, properties)
semantic actions (affordances like submit, cancel)
transitions (links between states)
Clients can read these descriptors and understand what the resources mean and what can be done, even without out-of-band documentation.
3. Anatomy of an ALPS document
ALPS is itself JSON or XML, with a top-level object that contains an array of descriptors.
Example in JSON form:
```json
{
  "alps": {
    "version": "1.0",
    "doc": { "value": "Profile for a Transcode Job" },
    "descriptor": [
      {
        "id": "transcode-job",
        "type": "semantic",
        "descriptor": [
          { "id": "id", "type": "semantic", "text": "Unique job identifier" },
          { "id": "state", "type": "semantic", "text": "queued | processing | completed | failed" },
          { "id": "source", "type": "semantic", "text": "link to source asset" },
          { "id": "destination", "type": "semantic", "text": "link to destination asset" }
        ]
      },
      {
        "id": "cancel",
        "type": "safe",
        "rt": "/transcodes/{id}",
        "text": "Transition job state to canceled"
      }
    ]
  }
}
```
Each descriptor can have:
an id — a unique name
a type — semantic, safe, unsafe, or idempotent (paralleling HTTP method semantics)
a text or doc — human-readable explanation
nested descriptor arrays for hierarchy
an rt (return type or target URI template)
4. Why ALPS exists
Before ALPS, every hypermedia system (HAL, Siren, Hydra, Collection+JSON) used slightly different ways to describe their domain semantics.
 That made it hard to create generic clients or share semantics across media types.
ALPS provides a media-type-independent vocabulary.
 You can write one ALPS profile, and use it for:
Siren documents
HAL representations
HTML forms
Collection+JSON
even non-JSON formats
It decouples meaning from format.
5. Relation to profiles (RFC 6906)
ALPS documents are typically published at a URI and referenced as profiles.
Example:
Content-Type: application/vnd.siren+json;profile="https://api.example.com/profiles/transcode-job"
Link: <https://api.example.com/profiles/transcode-job>; rel="profile"
At that URI (https://api.example.com/profiles/transcode-job), you serve an ALPS document that defines descriptors for the transcode-job resource.
That’s how clients discover what your resources mean.
6. Example for your transcoding domain
If you publish this at https://api.example.com/profiles/asset, you could define:
```json
{
  "alps": {
    "version": "1.0",
    "doc": { "value": "Profile for multimedia asset resources" },
    "descriptor": [
      { "id": "asset", "type": "semantic" },
      { "id": "id", "type": "semantic" },
      { "id": "state", "type": "semantic" },
      { "id": "container", "type": "semantic" },
      { "id": "video", "type": "semantic" },
      { "id": "audio", "type": "semantic" },
      { "id": "download", "type": "safe", "rt": "https://cdn.example.com/dl/{assetId}" }
    ]
  }
}
```
Clients could dereference this profile URI, parse the ALPS document, and automatically understand that:
id, state, container, etc. are standard semantics for an “asset.”
there’s a download action that is safe (like a GET).
7. ALPS and HATEOAS
ALPS and HATEOAS complement each other:
Siren or HAL gives you runtime hypermedia controls (links, actions).
ALPS gives you the vocabulary that describes what those controls mean.
A HAL or Siren document may include rel or class values that match ALPS descriptor IDs.
 A generic client can read those and know how to handle them.
So ALPS is the semantic dictionary behind your hypermedia.
8. Key advantages
Format-agnostic — works with any representation format.
Machine-readable — supports automated client behavior.
Human-understandable — includes short text documentation.
Reusable semantics — the same ALPS profile can describe multiple related APIs.
Integrates with profiles — fits perfectly with RFC 6906 (profile relation).
9. The essence
In one sentence:
ALPS provides a simple, machine-readable vocabulary of descriptors that define the semantics of your application’s resources and transitions — enabling generic clients to interpret and navigate your REST API without prior hardcoded knowledge.
It’s the “semantic spine” underneath hypermedia systems like HAL or Siren.
Would you like me to write the full ALPS profile documents for both your asset and transcode-job resources (that would live at your https://api.example.com/profiles/... URIs)?
2 replies


Sridhar Krishnamurthy
  Oct 8th at 7:28 PM
QUESTION
so can we say ALPS is like openAPI for hypermedia semantics ?
ANSWER
Yes — and that’s a very good way to think about it, as long as we clarify how it’s different.
Your intuition is right: ALPS and OpenAPI both describe APIs, but they live at different layers and serve different design philosophies.
Let’s unpack that cleanly.
1. Shared intent
Both OpenAPI and ALPS are about describing an API so that humans and machines can understand it.
 They both define what data exists and what operations are possible.
The difference lies in how and when those descriptions are used.
2. The core contrast
OpenAPI describes RPC-style or request/response APIs —
 what endpoints exist, what HTTP methods you call, what parameters and bodies they take, and what you get back.
It’s endpoint-centric and static:
 clients need the description before making calls.
ALPS, on the other hand, describes the semantics of resources and actions in a hypermedia system.
 It doesn’t enumerate endpoints or fixed workflows — it defines the meaning of the controls you discover dynamically through HATEOAS links and actions.
It’s semantic and dynamic:
 clients learn what they can do by reading the representation; ALPS just gives the vocabulary to interpret it.
3. In short
OpenAPI says:
“Here’s a list of endpoints you can call, and here’s how.”
ALPS says:
“When you see something called approve, or a transcode-job with state=processing, here’s what that means.”
So OpenAPI describes the surface contract; ALPS describes the semantic intent.
4. Example from your transcoding domain
OpenAPI would list something like:
POST /transcodes
PATCH /transcodes/{id}
GET /transcodes/{id}
and would document the fields, parameters, responses, and examples for each.
ALPS would define the meaning of descriptors such as:
source
destination
state
cancel
new-transcode
Then, when your Siren representation exposes a link or action with those identifiers, clients know what those affordances mean — even if they’ve never seen /transcodes before.
5. Analogy
If your API were a language:
OpenAPI is like a phrasebook — it lists sentences you can say and what they mean.
ALPS is like a dictionary — it defines the meaning of each word and concept so sentences (hypermedia controls) can be understood dynamically.
6. How they can coexist
You can use them together perfectly.
OpenAPI can describe how to call your API in a developer-centric way.
ALPS can describe what the API’s resources mean for hypermedia clients.
For example:
A developer building SDKs uses your OpenAPI spec.
A Siren client or browser-driven agent uses your ALPS profiles (via profile links and class descriptors) to interpret affordances at runtime.
7. Summary thought
Yes — ALPS is like OpenAPI for semantics, not syntax.
 It’s what you use when you want a REST system where:
endpoints are discovered, not documented,
actions are described semantically, not enumerated,
and clients navigate by meaning, not by URL.
You can think of it as the semantic twin of OpenAPI —
 where OpenAPI documents how to call, ALPS documents what it all means.


Sridhar Krishnamurthy
  Oct 8th at 7:44 PM
QUESTION
You had asked me earlier:
Would you like me to write the full ALPS profile documents for both your asset and transcode-job resources (that would live at your https://api.example.com/profiles/... URIs)?
Yes
ANSWER
Absolutely—here are two ready-to-serve ALPS profiles in JSON for your asset and transcode-job resources. They’re concise but complete enough for real clients to consume. You can host them at the indicated profile URIs and reference them via Content-Type ...;profile="<uri>" and Link: <...>; rel="profile".
Asset profile (save at https://api.example.com/profiles/asset)
{
  "alps": {
    "version": "1.0",
    "doc": { "value": "Profile for multimedia asset resources (source or destination), independent of representation format." },
    "descriptor": [
      {
        "id": "asset",
        "type": "semantic",
        "doc": { "value": "An addressable multimedia resource, local or external (e.g., S3)." },
        "descriptor": [
          { "id": "id", "type": "semantic", "doc": { "value": "Opaque asset identifier." } },
          { "id": "state", "type": "semantic", "doc": { "value": "Lifecycle state: pending | available | failed." } },
          { "id": "role", "type": "semantic", "doc": { "value": "Optional: source | destination | placeholder." } },
          { "id": "container", "type": "semantic", "doc": { "value": "File container (mp4, mov, mpeg-ts, etc.)." } },
          {
            "id": "video",
            "type": "semantic",
            "doc": { "value": "Video stream metadata." },
            "descriptor": [
              { "id": "video.codec", "type": "semantic" },
              { "id": "video.height", "type": "semantic" },
              { "id": "video.width", "type": "semantic" },
              { "id": "video.bitrate", "type": "semantic" },
              { "id": "video.framerate", "type": "semantic" }
            ]
          },
          {
            "id": "audio",
            "type": "semantic",
            "doc": { "value": "Audio stream metadata." },
            "descriptor": [
              { "id": "audio.codec", "type": "semantic" },
              { "id": "audio.channels", "type": "semantic" },
              { "id": "audio.sample_rate", "type": "semantic" },
              { "id": "audio.bitrate", "type": "semantic" }
            ]
          },
          { "id": "bytes", "type": "semantic", "doc": { "value": "Total size in bytes, when known." } },
          {
            "id": "storage",
            "type": "semantic",
            "doc": { "value": "Physical storage coordinates (provider/bucket/key…)" },
            "descriptor": [
              { "id": "storage.provider", "type": "semantic", "doc": { "value": "e.g., aws-s3, gcs, azure-blob." } },
              { "id": "storage.bucket", "type": "semantic" },
              { "id": "storage.key", "type": "semantic" },
              { "id": "storage.region", "type": "semantic" }
            ]
          }
        ]
      },

      { "id": "self", "type": "safe", "rt": "/assets/{id}", "doc": { "value": "Retrieve this asset representation (GET)." } },
      { "id": "collection", "type": "safe", "rt": "/assets", "doc": { "value": "Navigate to the assets collection (GET)." } },
      { "id": "download", "type": "safe", "rt": "https://cdn.example.com/dl/{id}", "doc": { "value": "Downloadable URL (GET). May be pre-signed and time-limited." } },

      { "id": "create-asset", "type": "unsafe", "rt": "/assets", "doc": { "value": "Create an asset wrapper (POST)." } },
      { "id": "update-asset", "type": "idempotent", "rt": "/assets/{id}", "doc": { "value": "Modify asset metadata (PATCH/PUT). Fields vary by representation." } },
      { "id": "delete-asset", "type": "idempotent", "rt": "/assets/{id}", "doc": { "value": "Remove an asset wrapper (DELETE). Content deletion depends on policy." } }
    ]
  }
}
Transcode-job profile (save at https://api.example.com/profiles/transcode-job)
{
  "alps": {
    "version": "1.0",
    "doc": { "value": "Profile for a transcoding job that transforms a source asset into a destination asset." },
    "descriptor": [
      {
        "id": "transcode-job",
        "type": "semantic",
        "doc": { "value": "A server-managed process with a lifecycle: queued → processing → completed | failed | canceled." },
        "descriptor": [
          { "id": "id", "type": "semantic" },
          { "id": "state", "type": "semantic", "doc": { "value": "queued | processing | completed | failed | canceled" } },
          { "id": "progress", "type": "semantic", "doc": { "value": "0–100 progress indicator (optional)." } },
          { "id": "created_at", "type": "semantic" },
          { "id": "updated_at", "type": "semantic" },

          {
            "id": "output_spec",
            "type": "semantic",
            "doc": { "value": "Requested output characteristics." },
            "descriptor": [
              { "id": "output.container", "type": "semantic" },
              {
                "id": "output.video",
                "type": "semantic",
                "descriptor": [
                  { "id": "output.video.codec", "type": "semantic" },
                  { "id": "output.video.height", "type": "semantic" },
                  { "id": "output.video.width", "type": "semantic" },
                  { "id": "output.video.bitrate", "type": "semantic" }
                ]
              },
              {
                "id": "output.audio",
                "type": "semantic",
                "descriptor": [
                  { "id": "output.audio.codec", "type": "semantic" },
                  { "id": "output.audio.bitrate", "type": "semantic" },
                  { "id": "output.audio.channels", "type": "semantic" }
                ]
              }
            ]
          },

          { "id": "source", "type": "semantic", "doc": { "value": "Link to source asset wrapper." } },
          { "id": "destination", "type": "semantic", "doc": { "value": "Link to destination asset wrapper." } }
        ]
      },

      { "id": "self", "type": "safe", "rt": "/transcodes/{id}", "doc": { "value": "Retrieve job representation (GET)." } },
      { "id": "collection", "type": "safe", "rt": "/transcodes", "doc": { "value": "Navigate to jobs collection (GET)." } },
      { "id": "logs", "type": "safe", "rt": "/transcodes/{id}/logs", "doc": { "value": "Retrieve job logs/diagnostics (GET)." } },

      { "id": "create-job", "type": "unsafe", "rt": "/transcodes", "doc": { "value": "Create a transcode job (POST). Body references source/destination assets and output_spec." } },
      { "id": "cancel", "type": "idempotent", "rt": "/transcodes/{id}", "doc": { "value": "Transition job to canceled (PATCH state=canceled). Only valid when queued/processing." } },
      { "id": "retry", "type": "unsafe", "rt": "/transcodes", "doc": { "value": "Create a new job using same or adjusted parameters (POST), typically after failure." } }
    ]
  }
}
How to reference these from responses (example headers)
Content-Type: application/vnd.siren+json;profile="https://api.example.com/profiles/asset"
Link: <https://api.example.com/profiles/asset>; rel="profile"

Content-Type: application/vnd.siren+json;profile="https://api.example.com/profiles/transcode-job"
Link: <https://api.example.com/profiles/transcode-job>; rel="profile"
If you want, I can also add minimal JSON Schemas that complement these ALPS profiles for structural validation (ALPS = semantics, JSON Schema = structure).


