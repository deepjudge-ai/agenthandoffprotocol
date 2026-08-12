# Agent Handoff Protocol v1 specification

**Status:** Draft 1.0

**Protocol version:** `v1`

**Last updated:** 2026-08-08

This document defines Agent Handoff Protocol (AHP) v1. AHP lets one agent
application transfer a user task to another agent application over HTTPS and
receive a URL for the prepared continuation.

The protocol was derived from the original DeepJudge implementation. This
specification describes the intended interoperable behavior, including the
complete sender and receiver responsibilities; incidental database, UI, and
agent-runtime choices are not part of the protocol.

For an introduction, see the [AHP README](README.md). For complete exchanges,
see [AHP examples](agent_handoff_protocol_examples.md).

## 1. Scope

AHP v1 standardizes:

- discovery of a version-specific handoff endpoint;
- authentication and deployment identity expectations;
- transfer of a task objective, selected conversation context, and resources;
- correlation of a continuing task across products and round trips;
- idempotent retry of one logical transfer; and
- return of a URL where the user can continue in the receiving product.

AHP v1 does not standardize credential issuance, user provisioning, tenant
mapping, agent implementation, search behavior, tool execution, model state,
streaming, or UI design.

The protocol is bilateral. The parties establish trust, client identifiers,
deployment origins, credentials, and user mapping out of band. There is no
open federation or global client registry in v1.

## 2. Requirements language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in
[BCP 14](https://www.rfc-editor.org/info/bcp14) when, and only when, they
appear in all capitals.

Unless stated otherwise:

- JSON names and string values are case-sensitive.
- HTTP field names are case-insensitive, as defined by HTTP.
- An absolute URL means an absolute URI using the `https` scheme.
- A receiver MAY accept input beyond the requirements in this specification,
  but a sender cannot rely on such extensions without bilateral agreement.

## 3. Terminology

| Term | Meaning |
| --- | --- |
| Agent application | A product or service that presents an agent experience to a user. |
| Sender | The application sending one handoff request. |
| Receiver | The application receiving that request and preparing the continuation. |
| Deployment | One independently configured instance of an agent application, identified by an HTTPS origin. |
| Transfer | One logical POST from a sender to a receiver. |
| Thread | A continuing user task that may cross applications more than once. |
| Handoff package | The objective, conversation sample, and resources in a transfer. |
| Client ID | A stable, bilaterally agreed identifier for an agent application. |
| Client origin | The HTTPS origin of the sending deployment. |
| Thread ID | An opaque identifier that correlates transfers belonging to the same task. |
| Idempotency key | A UUID that identifies one logical transfer and its retries. |
| Redirect URL | A receiver-generated URL for the prepared continuation. |
| Resource | One logical attachment, with a binary representation, a text representation, or both. |

Sender and receiver are roles, not permanent identities. If application B
returns a task to application A, B is the sender for that transfer and A is
the receiver.

## 4. Protocol overview

A successful transfer has two HTTP operations:

1. The sender performs an authenticated or public `GET` of the receiver's
   well-known discovery document and selects `v1.handoff_endpoint_url`.
2. The sender performs an authenticated `POST` to that URL, carrying the
   handoff headers and JSON package. The receiver returns a JSON object with
   `redirect_url`.

The authorization context and discovered endpoint determine the target user
and tenant. AHP deliberately has no body field that lets a sender name an
arbitrary receiver-side user or tenant.

The receiver MUST make the handoff durable before returning success. Agent
generation MAY continue asynchronously after that point, provided the URL is
already a valid place for the user to continue.

## 5. Versions and extensions

### 5.1 Protocol version

The selected member in the discovery document defines the protocol version.
AHP v1 uses the `v1` member. There is no separate version request header.

A discovery document MAY advertise several sibling versions. A sender MUST
choose the highest version it implements and MUST NOT infer support for an
unadvertised version. A sender that does not find a supported version MUST
stop without submitting a handoff.

An incompatible change requires a new discovery member and endpoint version.
Examples include removing a field, changing the meaning or type of a field,
or requiring a new content representation.

### 5.2 JSON extensibility

Objects MAY contain additional members. A receiver MUST ignore unrecognized
members whose names do not affect validation of a known discriminator. A
sender MUST ignore unrecognized members in discovery and success responses.

An unrecognized value of a `type` discriminator is not a harmless additional
member. The receiver MUST reject the containing message or resource unless it
has bilaterally negotiated support for that type.

Extensions MUST NOT weaken authentication, authorization, user consent,
idempotency, origin validation, or resource-fetching requirements.

### 5.3 MCP schema version

`context.conversation` reuses `SamplingMessage` from the
[Model Context Protocol schema dated 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/schema#samplingmessage).
AHP v1 is pinned to that dated schema. Later MCP revisions are not implicitly
part of AHP v1.

## 6. Deployment configuration and authentication

Before transferring a user, each side MUST configure at least:

- the other deployment's HTTPS discovery origin;
- the expected AHP client ID;
- an authentication method and credentials;
- the users, tenants, and scopes those credentials may act for; and
- trusted origins for discovered endpoints, remote resources, and redirect
  URLs.

AHP does not prescribe how credentials are obtained. OAuth 2.0 bearer tokens
in the `Authorization` header are RECOMMENDED; deployments MAY use another
HTTP authentication scheme by bilateral agreement. Bearer-token deployments
SHOULD follow [RFC 6750](https://www.rfc-editor.org/rfc/rfc6750.html).

The receiver MUST authenticate and authorize every handoff POST. If discovery
is user- or tenant-specific, the receiver MAY also require authentication for
the discovery GET. A sender MUST support authenticated discovery and MUST use
the credentials appropriate to the target user or service principal when the
receiver requires it.

The receiver MUST bind the authenticated principal to all of the following:

- the asserted `Handoff-Client-Id`;
- the asserted `Handoff-Client-Origin`;
- the target user and tenant selected by the endpoint and authorization
  context; and
- permission to create or resume a handoff thread.

`Handoff-Client-Id` and `Handoff-Client-Origin` are asserted routing metadata.
Neither is authentication proof. A receiver MUST NOT accept a request merely
because those fields match configured strings.

Credentials MUST NOT appear in a handoff JSON body, redirect URL, thread ID,
idempotency key, or ordinary log message. Long-lived or generally reusable
credentials MUST NOT be placed in remote URL queries. Narrowly scoped,
short-lived signed resource URLs are the exception described in
[Remote content](#remote-content).

## 7. Endpoint discovery

### 7.1 Discovery origin

The sender starts from an HTTPS origin configured out of band. It MUST NOT
derive an origin from untrusted user content or probe arbitrary Internet
hosts.

An origin is the scheme, host, and effective port. It has no user information,
path, query, or fragment. Origins MUST be compared using the ASCII
serialization rules in [RFC 6454](https://www.rfc-editor.org/rfc/rfc6454.html).

### 7.2 Well-known URI

AHP v1 uses this path at the root of the configured origin:

```text
/.well-known/agent-handoff-spec.json
```

The location follows the well-known URI model in
[RFC 8615](https://www.rfc-editor.org/rfc/rfc8615.html). Implementations MUST
support this path for AHP v1.

The sender sends:

```http
GET /.well-known/agent-handoff-spec.json HTTP/1.1
Host: receiver.example
Accept: application/json
Authorization: Bearer <credential>
```

The `Authorization` field is omitted only when the receiver documents public
discovery.

### 7.3 Discovery response

On success, the receiver MUST return `200 OK` and a JSON object using
`Content-Type: application/json`.

The minimum v1 document is:

```json
{
  "v1": {
    "handoff_endpoint_url": "https://receiver.example/api/ahp/v1/handoffs"
  }
}
```

| Member | Required | Type | Semantics |
| --- | --- | --- | --- |
| `v1` | Yes for v1 support | object | Configuration for AHP v1. |
| `v1.handoff_endpoint_url` | Yes | string | Absolute HTTPS URL that accepts the v1 handoff POST. |

The endpoint URL MUST NOT contain user information or a fragment. The
receiver MAY include a path or query when required for routing, although a
stable path without transient credentials is RECOMMENDED.

The sender MUST validate the discovered URL before use. By default, it MUST
require the URL to be same-origin with the configured discovery origin. A
cross-origin endpoint MAY be used only if that destination origin was
explicitly configured or allowlisted out of band.

The sender SHOULD reject discovery redirects to an untrusted origin. It MUST
apply the same trust checks to the final URL after redirects.

If the response varies by authenticated principal, user, or tenant, the
receiver SHOULD return `Cache-Control: private, no-store` or appropriate
`Vary` metadata. A sender MUST NOT reuse an authenticated discovery result
for a different principal unless the receiver explicitly declares it safe.

### 7.4 Discovery failures

The sender MUST NOT submit a handoff when discovery:

- returns a non-success status;
- exceeds the sender's timeout;
- is not valid JSON;
- omits `v1.handoff_endpoint_url`;
- returns a non-HTTPS or otherwise untrusted endpoint; or
- does not advertise a version the sender implements.

Discovery is safe to retry as an HTTP `GET`. Retries SHOULD use bounded
exponential backoff.

## 8. Submitting a handoff

### 8.1 HTTP request

The sender performs an HTTPS `POST` to the discovered v1 endpoint.

```http
POST /api/ahp/v1/handoffs HTTP/1.1
Host: receiver.example
Authorization: Bearer <credential>
Content-Type: application/json
Accept: application/json, application/problem+json
Handoff-Thread-Id: 018f62c4-8f67-7d2a-a751-57e7d2ad59cf
Handoff-Client-Id: sender
Handoff-Client-Origin: https://sender.example
Idempotency-Key: 2e6e1ec5-3f55-4e16-a4de-ef8d54154cce

{...}
```

JSON request and response documents MUST be encoded as UTF-8. The sender MUST
send `Content-Type: application/json`.

### 8.2 Required request fields

| HTTP field | Value | Requirements |
| --- | --- | --- |
| `Authorization` | Deployment-selected credentials | REQUIRED unless another bilaterally agreed HTTP authentication scheme supplies credentials. |
| `Content-Type` | `application/json` | REQUIRED. |
| `Accept` | Includes `application/json` | RECOMMENDED; including `application/problem+json` is also RECOMMENDED. |
| `Handoff-Thread-Id` | Opaque string | REQUIRED; stable across transfers in the same task. |
| `Handoff-Client-Id` | Bilaterally agreed client ID | REQUIRED. |
| `Handoff-Client-Origin` | ASCII serialization of the sender's HTTPS origin | REQUIRED; no trailing slash. |
| `Idempotency-Key` | Canonical UUID text | REQUIRED; new per logical transfer, unchanged across retries. |

HTTP field names are case-insensitive. Their values are case-sensitive unless
the underlying type, such as an origin host, defines normalization.

#### Handoff-Client-Id

A v1 client ID MUST be 1 to 128 characters drawn from ASCII letters, digits,
`.`, `_`, and `-`. Lowercase identifiers are RECOMMENDED. The value is opaque
outside the bilateral relationship; v1 defines no public registry.

#### Handoff-Client-Origin

The value MUST be an RFC 6454 ASCII-serialized HTTPS origin. It MUST NOT
contain a path, trailing slash, query, fragment, or user information. The
receiver MUST normalize and compare origins structurally rather than using an
unsafe prefix match.

#### Handoff-Thread-Id

The value MUST be 1 to 255 visible ASCII characters and MUST NOT contain
whitespace or control characters. It is opaque and case-sensitive. A UUID is
RECOMMENDED for a newly initiated thread.

The thread ID is not a secret or authorization token.

#### Idempotency-Key

The value MUST be a canonical textual UUID, for example a randomly generated
UUID v4. AHP v1 uses the bare UUID value without Structured Fields quoting.
The semantics are defined in [Section 11](#11-idempotency-and-retries).

### 8.3 Request body

The request body has exactly one required top-level member:

```typescript
interface HandoffRequest {
  context: HandoffContext;
}

interface HandoffContext {
  objective: string;
  conversation: SamplingMessage[];
  resources: HandoffResource[];
}
```

`context`, `objective`, `conversation`, and `resources` MUST be present. The
two arrays MAY be empty. `null` is not a substitute for either array.

### 8.4 Objective

`context.objective` is the task the receiving agent should continue. It MUST
be a non-empty JSON string after trimming surrounding whitespace.

The sender SHOULD make the objective:

- stand alone without requiring the entire transcript;
- state the desired outcome and material constraints;
- distinguish completed work from remaining work when relevant; and
- avoid product-specific tool instructions the receiver cannot follow.

The receiver MUST present the objective to its agent as user-authorized task
input, below receiver-side system and policy instructions. The transcript is
supporting context and MUST NOT silently override the objective.

### 8.5 Conversation

`context.conversation` is a JSON array of MCP `SamplingMessage` objects in
chronological order. A sender MAY provide all or only a useful subset of the
conversation, but MUST preserve the relative order of messages it includes.

Each message has:

| Member | Required | Type | Semantics |
| --- | --- | --- | --- |
| `role` | Yes | `"user"` or `"assistant"` | Original conversational role. |
| `content` | Yes | MCP `SamplingMessageContentBlock` or an array of those blocks | Message content under MCP 2025-11-25. |
| `_meta` | No | object | MCP metadata; omit unless its meaning and disclosure are intentional. |

A sender SHOULD encode `content` as an array even when it contains one block,
because arrays avoid a shape change when another block is added. A receiver
MUST accept both the single-block and array forms defined by MCP.

An AHP v1 receiver MUST support MCP text blocks:

```json
{
  "type": "text",
  "text": "Please compare the termination clauses."
}
```

MCP 2025-11-25 also defines image, audio, tool-use, and tool-result blocks for
sampling messages. A receiver MAY support those richer blocks. If it cannot
process a received block safely and faithfully, it MUST reject the handoff
with `422 Unprocessable Content` before creating the continuation; it MUST NOT
silently pretend the context was fully imported.

The sender MUST apply data minimization before serialization. It MUST NOT
include:

- hidden chain-of-thought or private model reasoning;
- system or developer prompts;
- credentials, access tokens, cookies, or secret tool inputs;
- internal handoff bookkeeping or copies of earlier imported handoffs;
- internal notifications with no value to the receiving user; or
- private tool traces merely because they appear in local history.

Tool-use and tool-result blocks SHOULD be omitted unless the receiver can
understand them and the user approved their disclosure. Human-visible tool
outcomes can instead be summarized in text or attached as resources.

### 8.6 Resources

`context.resources` is an array of zero to 5,000 `HandoffResource` objects.
Resource order has no protocol meaning, and names need not be unique.

```typescript
interface HandoffResource {
  name: string;
  description?: string | null;
  blob?: RemoteContent | InlineBlobContent | null;
  text?: RemoteContent | InlineTextContent | null;
}

interface InlineTextContent {
  type: "inline_text";
  mime_type: string;
  data: string;
}

interface InlineBlobContent {
  type: "inline_blob";
  mime_type: string;
  data: string; // RFC 4648 base64
}

interface RemoteContent {
  type: "remote";
  mime_type?: string | null;
  url: string;
}
```

#### Resource object rules

- `name` is REQUIRED and MUST be a non-empty display name.
- `description` is OPTIONAL agent-facing context, metadata, provenance, or a
  short explanation of relevance. It is untrusted content.
- At least one of `blob` or `text` MUST be non-null.
- `blob` and `text` MAY both be present. They are representations of the same
  logical resource, not two attachments.
- A receiver MAY use one or both representations according to its needs, but
  MUST keep them associated as one resource.
- A filename is display metadata, not a path. Receivers MUST sanitize it and
  MUST NOT honor directory traversal components.

The resource envelope uses `mime_type` in snake case. MCP content blocks use
MCP's `mimeType` spelling. Implementers MUST preserve this distinction.

#### Inline text

For `type: "inline_text"`, `data` is a JSON string and therefore Unicode text.
The sender SHOULD use a registered textual media type and SHOULD include a
charset when ambiguity is possible, for example `text/plain; charset=utf-8`.

The `data` string MUST NOT exceed 10,000,000 characters.

#### Inline blob

For `type: "inline_blob"`, `data` MUST be the standard base64 encoding from
[RFC 4648 Section 4](https://www.rfc-editor.org/rfc/rfc4648.html#section-4),
including padding when required. It MUST NOT contain a data-URL prefix,
whitespace, line breaks, or non-alphabet characters.

`mime_type` describes the decoded bytes, not the base64 text. The `data`
string MUST NOT exceed 10,000,000 characters. Receivers MUST enforce a limit
on decoded bytes as well.

#### Remote content

For `type: "remote"`, `url` MUST be an absolute HTTPS URL without user
information or a fragment and MUST NOT exceed 2,083 characters. `mime_type`
is OPTIONAL. When supplied, it is the sender's assertion; the receiver MUST
still validate the fetched response.

The sender MUST ensure the receiver can dereference the URL using one of:

- receiver credentials configured for the sender deployment;
- a short-lived, narrowly scoped signed URL; or
- another bilaterally agreed authorization mechanism.

The credential used by the sender to call the receiver MUST NOT simply be
forwarded back to the URL host. Tokens must be audience-restricted.

Before returning success, the receiver MUST validate and either materialize
the remote representation or create a durable, authorized reference that
will remain usable for the promised continuation. Materialization is
RECOMMENDED. The receiver MUST fail the whole handoff if a required remote
representation cannot be fetched or validated.

When fetching remote content, the receiver MUST:

- allow only configured HTTPS origins by default;
- resolve and re-check every redirect target;
- prevent access to loopback, link-local, private, cloud-metadata, and other
  internal addresses unless explicitly required and allowlisted;
- apply connection, response, and total timeouts;
- limit redirects, compressed and decoded size, and decompression ratio;
- validate media type and process content as untrusted;
- scan or quarantine files according to receiver policy; and
- avoid logging content or credential-bearing URLs.

HTTP URLs MAY be accepted only in an explicitly configured local development
environment and MUST NOT be used for production AHP traffic.

#### Choosing inline or remote

Senders SHOULD use inline representations for small, stable payloads and
remote representations for large or already-hosted content. A sender SHOULD
include extracted `text` alongside a binary `blob` when that materially helps
the receiving agent and the disclosure is user-approved.

### 8.7 Request limits

The v1 schema ceilings are:

- at most 5,000 resources per transfer; and
- at most 10,000,000 characters in each inline `data` member; and
- at most 2,083 characters in each remote resource URL.

These are protocol ceilings, not a promise that every deployment accepts a
request near that size. Receivers MUST document operational request, resource,
message, and decoded-content limits. Senders SHOULD keep context small and use
remote resources for large artifacts. A receiver rejecting an oversized
request SHOULD use `413 Content Too Large`.

## 9. Successful response

After durably accepting the transfer, the receiver MUST return `200 OK` with
`Content-Type: application/json` and this body:

```typescript
interface HandoffResponse {
  redirect_url: string;
}
```

Example:

```json
{
  "redirect_url": "https://receiver.example/workspaces/continue/7d42d2f1"
}
```

`redirect_url` MUST be an absolute HTTPS URL without user information. It
MUST identify a continuation the authenticated target user is allowed to
open. It SHOULD be on a receiver origin configured by the sender. It MUST NOT
contain bearer credentials or other reusable secrets.

The URL is opaque to the sender. The sender MUST NOT parse receiver-specific
conversation, tenant, or session identifiers from it.

The sender MUST validate the URL's scheme and origin before presenting or
navigating to it. Cross-origin redirects are allowed only when explicitly
trusted for that receiver integration. The sender SHOULD preserve user agency
by making the destination clear before navigation.

The response SHOULD include `Cache-Control: no-store`. A retry with the same
idempotency key and the same logical request MUST return the same effective
redirect URL.

A conforming receiver returns `200`. A sender MAY accept another `2xx` status
only when the response has a valid `HandoffResponse` and the sender's product
has explicitly chosen to support it.

## 10. Receiver processing model

For a new idempotency key, the receiver performs these conceptual steps:

1. Authenticate the HTTP request and resolve the target user and tenant.
2. Validate that the authenticated identity may assert the client ID and
   client origin.
3. Validate headers, JSON shape, content types, and configured limits.
4. Acquire concurrency control for the scoped idempotency key.
5. Look up a prior result and compare its request fingerprint.
6. Resolve the thread within the authenticated user, client, and deployment
   scope, creating a new local continuation if none exists.
7. Fetch, validate, and materialize resources as an all-or-nothing operation.
8. Store the objective, selected conversation, resources, thread mapping,
   request fingerprint, and redirect URL.
9. Make the handoff visible to the receiver's agent and UI.
10. Commit the continuation and idempotency result atomically.
11. Return the redirect URL.

Steps may be implemented differently, but these externally visible
invariants are REQUIRED:

- A failed handoff MUST NOT leave a successful idempotency result.
- A success response MUST NOT race ahead of durable thread and resource state.
- Two concurrent requests with the same key MUST produce at most one set of
  side effects.
- A partial resource set MUST NOT be presented as a complete successful
  handoff.
- The receiver MUST apply its own authorization, compliance, retention, and
  agent policies to imported context.

The receiver SHOULD start useful agent work from the objective. It SHOULD use
the conversation as context, not as an instruction to repeat already
completed actions. If required information is missing, the receiving agent
may ask the user to clarify.

## 11. Idempotency and retries

HTTP `POST` is not inherently idempotent. `Idempotency-Key` makes one AHP
transfer safe to retry after a timeout, connection loss, or retryable server
failure.

### 11.1 Sender requirements

The sender MUST:

- generate a new UUID for every new logical transfer;
- use the same key for every retry of that transfer;
- keep the endpoint, handoff headers, target authorization identity, and JSON
  body semantically unchanged across those retries; and
- never reuse the key for a later transfer, including a later transfer on the
  same thread.

### 11.2 Receiver scope

The receiver MUST scope an idempotency key by at least:

- receiver deployment and target tenant;
- authenticated target user or service-principal operation;
- authenticated sender identity;
- `Handoff-Client-Id`; and
- `Handoff-Client-Origin` or the resolved bilateral integration.

This prevents one user or deployment from observing or colliding with
another's result.

### 11.3 Request fingerprint

The receiver MUST store a fingerprint of the logical request with the
idempotency result. The fingerprint MUST cover:

- the selected AHP version and endpoint operation;
- thread ID, client ID, and client origin;
- the resolved target identity and integration; and
- the complete handoff JSON body.

The fingerprint MUST NOT include volatile transport fields or credentials.
An implementation may hash canonical JSON or compare a validated semantic
representation.

If the same scoped key is replayed with the same fingerprint, the receiver
MUST return the stored success response without creating resources, messages,
threads, or agent runs again.

If the same scoped key is replayed with a different fingerprint, the receiver
MUST return `409 Conflict` and MUST NOT process the new request.

### 11.4 Concurrency and retention

Concurrent requests using the same scoped key MUST be serialized. Once the
first succeeds, all matching duplicates MUST receive the same effective
success response. Implementations may use a transactional lock, unique
constraint, or equivalent mechanism.

A receiver MUST retain a successful idempotency result for at least 24 hours
and SHOULD retain it for as long as the associated handoff record. The
receiver MUST document a longer or finite retention policy when clients may
retry outside that baseline window.

### 11.5 Retry policy

When the transfer outcome is unknown or retryable, a sender SHOULD retry with
bounded exponential backoff and jitter. It SHOULD honor `Retry-After`.

The sender MAY retry the unchanged request after:

- a connection failure or timeout with an unknown outcome;
- `408 Request Timeout`;
- `429 Too Many Requests`; or
- a transient `5xx` response.

The sender SHOULD NOT automatically retry other `4xx` responses. After
correcting a rejected request, it MUST use a new idempotency key because that
is a new logical transfer.

## 12. Errors

A receiver MUST use HTTP status semantics consistently. The following mapping
is RECOMMENDED:

| Status | Condition |
| --- | --- |
| `400 Bad Request` | Malformed HTTP, malformed JSON, or invalid header syntax. |
| `401 Unauthorized` | Missing, expired, or invalid authentication. |
| `403 Forbidden` | Authenticated caller lacks permission for the target user, tenant, or operation. |
| `409 Conflict` | Same idempotency key with a different request, wrong return deployment, or incompatible thread ownership. |
| `413 Content Too Large` | Request, inline representation, decoded resource, or resource count exceeds an operational limit. |
| `415 Unsupported Media Type` | Request is not JSON. |
| `422 Unprocessable Content` | Well-formed request violates the AHP schema, asserts an unknown client/origin, contains an unsupported content block, or has a resource that cannot be processed. |
| `429 Too Many Requests` | Rate limit exceeded. |
| `502 Bad Gateway` | A required remote dependency returned an invalid or failed response. |
| `504 Gateway Timeout` | Timed out fetching a required remote dependency. |
| `5xx` | Unexpected receiver failure. |

For machine-readable errors, receivers SHOULD use
[RFC 9457 Problem Details](https://www.rfc-editor.org/rfc/rfc9457.html) with
`Content-Type: application/problem+json`. Error `detail` is for humans;
senders MUST use the status and stable problem type or extension members for
logic rather than parsing prose.

An error response MUST NOT expose access tokens, signed resource URLs,
conversation content, resource content, internal stack traces, database IDs,
or configuration details. A `401` response using an HTTP authentication
scheme SHOULD include the appropriate `WWW-Authenticate` field.

## 13. Security and privacy

### 13.1 User authorization and data minimization

The sending product MUST obtain an affirmative user action before a handoff.
The user SHOULD be able to inspect the destination and remove resources before
submission. A sender MUST transfer only context the user is allowed to share.

The sender SHOULD select the minimum conversation and resource set that lets
the receiver make progress. It MUST apply the source product's tenant,
matter, document, and data-loss-prevention policies before transfer.

### 13.2 Authentication is not header matching

Client ID and origin checks are defense in depth and deployment routing. The
receiver MUST authenticate the caller and verify that the caller is allowed
to assert those values. It MUST reject confused-deputy attempts where valid
credentials for one client, origin, tenant, or user are used with another.

Access tokens SHOULD be audience-restricted and least-privileged. A receiver
MUST enforce tenant and user boundaries on thread lookup and idempotency
records.

### 13.3 Prompt injection and instruction boundaries

Every transferred string and byte sequence is untrusted input, including the
objective, transcript, resource name, description, extracted text, and remote
content. The receiver MUST NOT treat any of them as receiver-side system or
developer instructions.

The receiver SHOULD clearly delimit imported conversation and resources in
the prompt or agent state. Its own safety, authorization, compliance, and tool
policies remain authoritative. AHP transfers no source-side privilege.

### 13.4 Hidden state and secrets

Senders MUST NOT serialize hidden reasoning, system prompts, developer
messages, private policy text, credentials, cookies, encryption keys, or
unnecessary tool inputs and outputs. Receivers MUST assume `_meta` and tool
content may contain sensitive data and apply the same controls as to the rest
of the payload.

### 13.5 Remote-resource security

Remote retrieval creates SSRF, DNS rebinding, redirect, decompression bomb,
malware, content-type confusion, and time-of-check/time-of-use risks. The
controls in [Remote content](#remote-content) are REQUIRED. In particular,
allowing all HTTPS destinations is not a sufficient SSRF defense.

If signed URLs are used, they SHOULD be single-resource, read-only,
short-lived, and redacted from logs. Materializing content before success
reduces link-expiry and content-mutation risks.

### 13.6 Redirect security

The sender MUST treat `redirect_url` as untrusted receiver input. It MUST
allowlist schemes and origins, reject user information and dangerous schemes,
and avoid forwarding API credentials during browser navigation. Receiver
implementations MUST prevent their continuation endpoint from becoming an
open redirect.

### 13.7 Logging and observability

Ordinary logs SHOULD contain safe identifiers and outcome metadata, not
objectives, transcript text, filenames, descriptions, bodies, bearer tokens,
or signed URLs. Debug details and user-facing error details SHOULD be separate
so internal topology is not leaked to the sender or user.

Audit logs SHOULD record who initiated and received a transfer, the two
deployments, time, thread and transfer identifiers, resource count, result,
and policy decision. If thread IDs or idempotency keys are considered
sensitive in a deployment, audit systems SHOULD hash or access-control them.

### 13.8 Retention and deletion

Both parties MUST apply disclosed retention policies to handoff records and
materialized resources. Disabling an integration SHOULD prevent new transfers
without corrupting historical conversations. Deleting a user or tenant MUST
not leave accessible orphaned handoff resources.

### 13.9 Denial of service

Receivers SHOULD rate-limit discovery and transfer endpoints, bound JSON
parsing and base64 decoding, cap concurrent remote fetches, and apply quotas
per authenticated client and user. Validation that can reject a request
without remote I/O SHOULD occur before resource fetching.

## 14. Thread semantics and round trips

The initiator of a new cross-application task SHOULD generate the thread ID.
Every genuine transfer of that task reuses the same thread ID and uses a new
idempotency key.

The receiver MUST scope a thread by at least the target user, authenticated
remote client, and remote deployment or bilateral integration. Equal thread
strings from different users or deployments MUST NOT merge conversations.

When a received task is returned to the deployment from which it came, the
sender MUST:

- target the original integration or deployment;
- reuse the original thread ID;
- create a new idempotency key; and
- send a fresh, minimized snapshot of the useful current context.

Each application SHOULD map both sent and received transfers to its local
thread or workspace. This is necessary for the original sender to resume its
local task when the partner returns it.

Protocol bookkeeping messages and previously imported transcript copies
SHOULD be excluded from later snapshots. This prevents recursive context
growth and accidental disclosure of internal metadata.

If the scoped thread mapping no longer exists, the receiver MAY create a new
local continuation under the same thread ID. It SHOULD make the loss of prior
local context clear to the user or agent when that affects the task.

## 15. Operational guidance

- Discovery and transfer clients SHOULD use explicit timeouts. A 10-second
  discovery timeout and a 30-second transfer timeout are reasonable starting
  points, but remote resource materialization may require a documented longer
  limit.
- Receivers SHOULD commit handoff records and resources in one transaction or
  an equivalent recoverable workflow.
- Agent work that runs after the HTTP response SHOULD be observable and
  recoverable independently of transfer idempotency.
- Senders SHOULD persist the thread ID, idempotency key, request fingerprint,
  destination, and validated redirect until the transfer outcome is known.
- Metrics SHOULD distinguish new threads, resumed threads, successes,
  validation failures, authentication failures, remote-resource failures,
  duplicate retries, and key conflicts without recording user content.
- Clocks are not part of correctness in v1; neither transfer timestamps nor
  expiry timestamps appear in the wire request.

## 16. Conformance

### 16.1 AHP v1 sender

A conforming sender MUST:

- implement v1 discovery at a configured origin;
- validate the discovered endpoint and authenticate as required;
- emit all required headers and the required JSON body;
- minimize and correctly encode conversation context and resources;
- implement thread reuse and idempotent retries;
- reject invalid success responses and unsafe redirect URLs; and
- satisfy the sender-side security requirements in this document.

### 16.2 AHP v1 receiver

A conforming receiver MUST:

- advertise a valid v1 endpoint at the historical well-known path;
- authenticate and authorize transfer requests;
- bind authenticated identity to client ID, origin, target user, and tenant;
- accept the v1 JSON model and MCP text-message baseline;
- support inline text, inline blob, and secure remote resource processing;
- implement scoped, concurrency-safe idempotency with conflict detection;
- implement scoped thread continuation and round-trip reuse;
- make the continuation durable before returning a safe redirect URL; and
- satisfy the receiver-side security requirements in this document.

### 16.3 Minimum interoperability tests

An implementation claiming v1 conformance SHOULD test at least:

1. authenticated and public discovery, as supported;
2. minimal handoff with an empty conversation and resource list;
3. multiple text messages using both single and array `content` forms;
4. inline text and valid/invalid inline base64 resources;
5. a resource carrying both text and blob representations;
6. authorized remote content, disallowed origins, redirects, timeouts, and
   size limits;
7. retry after an unknown outcome with the same key;
8. concurrent duplicate requests;
9. same-key/same-request replay and same-key/different-request conflict;
10. a new transfer on the same thread with a new key;
11. a full A to B to A round trip;
12. isolation of equal thread IDs and keys across users and deployments;
13. invalid client ID/origin assertions;
14. unsupported MCP content blocks; and
15. unsafe redirect URLs.

## 17. References

Normative and informative foundations used by AHP v1:

- [BCP 14: Requirements language](https://www.rfc-editor.org/info/bcp14)
- [RFC 6454: The Web Origin Concept](https://www.rfc-editor.org/rfc/rfc6454.html)
- [RFC 6750: OAuth 2.0 Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750.html)
- [RFC 8259: The JavaScript Object Notation Data Interchange Format](https://www.rfc-editor.org/rfc/rfc8259.html)
- [RFC 8615: Well-Known Uniform Resource Identifiers](https://www.rfc-editor.org/rfc/rfc8615.html)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457.html)
- [RFC 9562: Universally Unique IDentifiers](https://www.rfc-editor.org/rfc/rfc9562.html)
- [MCP schema, 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25/schema)

The work-in-progress IETF
[Idempotency-Key HTTP Header Field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
is useful background but is not normative for AHP v1. In particular, AHP v1
uses an unquoted canonical UUID field value for compatibility with deployed
implementations.
