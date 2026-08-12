# Agent Handoff Protocol v1 examples

This document contains worked AHP v1 exchanges and implementation sketches.
The [AHP v1 specification](agent_handoff_protocol_spec.md) is normative; these
examples are explanatory.

All example domains use the reserved `.example` suffix. Tokens and identifiers
are fictional.

## Example relationship

The examples use two applications:

| Property | Alpha | Beta |
| --- | --- | --- |
| Product client ID | `alpha-agent` | `beta-agent` |
| Deployment origin | `https://alpha.example` | `https://beta.example` |
| Handoff endpoint | `https://alpha.example/api/ahp/v1/handoffs` | `https://beta.example/api/ahp/v1/handoffs` |

Alpha and Beta have already configured credentials, user mapping, allowed
origins, and the two client IDs out of band.

The recurring thread ID is:

```text
b66d4ea0-77d9-45d7-a58f-8c04dcb95692
```

## 1. Discover Beta's endpoint

Alpha begins with Beta's configured origin and the credentials for the user
who requested the handoff.

```http
GET /.well-known/agent-handoff-spec.json HTTP/1.1
Host: beta.example
Accept: application/json
Authorization: Bearer eyJhbGciOi...redacted
```

Beta returns a user- or tenant-appropriate endpoint:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: private, no-store

{
  "v1": {
    "handoff_endpoint_url": "https://beta.example/api/ahp/v1/handoffs"
  }
}
```

Alpha checks that the result is valid JSON, contains `v1`, uses HTTPS, has no
userinfo or fragment, and is on the configured Beta origin.

The same discovery with `curl` looks like this:

```bash
curl --fail-with-body \
  --header "Authorization: Bearer $BETA_ACCESS_TOKEN" \
  --header "Accept: application/json" \
  https://beta.example/.well-known/agent-handoff-spec.json
```

## 2. Minimal handoff

A minimal handoff can contain only an objective. The conversation and resource
arrays are still required, but may be empty.

```http
POST /api/ahp/v1/handoffs HTTP/1.1
Host: beta.example
Authorization: Bearer eyJhbGciOi...redacted
Content-Type: application/json
Accept: application/json, application/problem+json
Handoff-Thread-Id: b66d4ea0-77d9-45d7-a58f-8c04dcb95692
Handoff-Client-Id: alpha-agent
Handoff-Client-Origin: https://alpha.example
Idempotency-Key: 6a3c73ee-3996-4ec7-9c5c-9b35a4955d0f

{
  "context": {
    "objective": "Compare the termination provisions in the selected agreements and explain the material differences.",
    "conversation": [],
    "resources": []
  }
}
```

Beta creates or resumes the scoped thread and returns a ready continuation:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "redirect_url": "https://beta.example/workspaces/continue/4f1c3128"
}
```

## 3. Conversation context

Conversation entries use MCP `SamplingMessage`. `content` may be one content
block or an array. The array form is recommended and shown here.

```json
{
  "context": {
    "objective": "Find the change-of-control termination right and assess whether the buyer can invoke it after signing.",
    "conversation": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "Which agreement gives the buyer the broadest termination right?"
          }
        ]
      },
      {
        "role": "assistant",
        "content": [
          {
            "type": "text",
            "text": "The Northwind agreement appears broadest, but I have not yet checked its change-of-control limitation."
          }
        ]
      },
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "Please verify that limitation in Beta."
          }
        ]
      }
    ],
    "resources": []
  }
}
```

The objective says what remains to do. The conversation explains why that is
the next step without forcing Beta to infer it from the last message.

## 4. Inline text resource

This resource carries a small user note directly in the JSON request:

```json
{
  "name": "review-notes.txt",
  "description": "The user's notes from the first-pass review.",
  "text": {
    "type": "inline_text",
    "mime_type": "text/plain; charset=utf-8",
    "data": "Focus on sections 8.1(c), 8.2(a), and the defined term Change of Control."
  }
}
```

`name` is a display name. Beta sanitizes it instead of interpreting it as a
filesystem path.

## 5. Inline binary resource

`inline_blob.data` is standard RFC 4648 base64 without a data-URL prefix. The
following payload decodes to the bytes `Exhibit A\n`:

```json
{
  "name": "exhibit-a.txt",
  "description": "A small exhibit attached by the user.",
  "blob": {
    "type": "inline_blob",
    "mime_type": "text/plain",
    "data": "RXhoaWJpdCBBCg=="
  }
}
```

For a real PDF or other large binary, a remote representation is usually a
better choice.

## 6. One resource with binary and text representations

Alpha can provide both the original document and extracted text. They remain
one logical resource:

```json
{
  "name": "northwind-purchase-agreement.pdf",
  "description": "Purchase agreement selected by the user. The first review flagged sections 8.1(c) and 8.2(a).",
  "blob": {
    "type": "remote",
    "mime_type": "application/pdf",
    "url": "https://alpha.example/api/resources/01J4NW/content"
  },
  "text": {
    "type": "remote",
    "mime_type": "text/plain; charset=utf-8",
    "url": "https://alpha.example/api/resources/01J4NW/text"
  }
}
```

Beta may use the extracted text for its agent and retain the PDF for preview,
citation, or download. It does not create two unrelated attachments.

The remote URLs must be accessible to Beta. For example, Beta might fetch
them with separately configured Beta-to-Alpha credentials:

```http
GET /api/resources/01J4NW/text HTTP/1.1
Host: alpha.example
Authorization: Bearer <beta-to-alpha-resource-token>
Accept: text/plain
```

That credential is not Alpha's token for calling Beta. Tokens should be
audience-restricted and are never copied from the inbound handoff request.

Beta validates the final URL after redirects, enforces size and time limits,
checks the media type, and materializes the resource before acknowledging the
handoff.

## 7. Complete handoff with mixed resources

The body below combines conversation context, inline text, and a dual remote
resource. It is valid JSON and can be saved as `handoff.json`.

```json
{
  "context": {
    "objective": "Verify whether the buyer may terminate after a change of control and produce a short answer with section references.",
    "conversation": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "Which agreement gives the buyer the broadest termination right?"
          }
        ]
      },
      {
        "role": "assistant",
        "content": [
          {
            "type": "text",
            "text": "The Northwind agreement appears broadest, subject to checking its change-of-control limitation."
          }
        ]
      }
    ],
    "resources": [
      {
        "name": "review-notes.txt",
        "description": "The user's notes from the first-pass review.",
        "text": {
          "type": "inline_text",
          "mime_type": "text/plain; charset=utf-8",
          "data": "Focus on sections 8.1(c), 8.2(a), and the defined term Change of Control."
        }
      },
      {
        "name": "northwind-purchase-agreement.pdf",
        "description": "Purchase agreement selected by the user.",
        "blob": {
          "type": "remote",
          "mime_type": "application/pdf",
          "url": "https://alpha.example/api/resources/01J4NW/content"
        },
        "text": {
          "type": "remote",
          "mime_type": "text/plain; charset=utf-8",
          "url": "https://alpha.example/api/resources/01J4NW/text"
        }
      }
    ]
  }
}
```

Alpha submits it with:

```bash
curl --fail-with-body \
  --request POST \
  --header "Authorization: Bearer $BETA_ACCESS_TOKEN" \
  --header "Content-Type: application/json" \
  --header "Accept: application/json, application/problem+json" \
  --header "Handoff-Thread-Id: b66d4ea0-77d9-45d7-a58f-8c04dcb95692" \
  --header "Handoff-Client-Id: alpha-agent" \
  --header "Handoff-Client-Origin: https://alpha.example" \
  --header "Idempotency-Key: 7d1e3ef1-68c3-42f8-96a4-7d9a2f741252" \
  --data-binary @handoff.json \
  https://beta.example/api/ahp/v1/handoffs
```

## 8. Unknown outcome and safe retry

Suppose Alpha sends transfer key
`7d1e3ef1-68c3-42f8-96a4-7d9a2f741252`. Beta commits the continuation and
starts returning `200`, but the connection closes before Alpha receives the
body. Alpha cannot know whether the operation happened.

Alpha retries:

- the same endpoint;
- the same authenticated target identity;
- the same thread ID, client ID, and client origin;
- the same JSON body; and
- the same idempotency key.

Beta finds the stored matching fingerprint and returns the original result
without creating another thread, resource set, message, or agent run:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "redirect_url": "https://beta.example/workspaces/continue/4f1c3128"
}
```

The retry is not a new handoff. Generating a new idempotency key after the
timeout would risk duplicate work.

## 9. Same key with a different request

Now suppose a bug reuses that idempotency key but changes the objective or
resource list. Beta compares the request fingerprint and rejects it:

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json
Cache-Control: no-store

{
  "type": "https://beta.example/problems/idempotency-key-reused",
  "title": "Idempotency key already used",
  "status": 409,
  "detail": "The key belongs to a different handoff request. Generate a new key for a new transfer."
}
```

Alpha must not overwrite the earlier result. If the changed request is
intentional, Alpha generates a new key and submits it as a new transfer.

## 10. A second transfer on the same thread

After the user does more work in Alpha, Alpha may send a genuine second
handoff to Beta. It reuses the thread ID but generates a new transfer key:

| Value | First transfer | Second transfer |
| --- | --- | --- |
| `Handoff-Thread-Id` | `b66d4ea0-77d9-45d7-a58f-8c04dcb95692` | `b66d4ea0-77d9-45d7-a58f-8c04dcb95692` |
| `Idempotency-Key` | `7d1e3ef1-68c3-42f8-96a4-7d9a2f741252` | `ba92ae22-a83b-4740-a30a-632738663c51` |
| Meaning | One logical transfer and its retries | A new logical transfer in the existing task |

Beta resolves the scoped thread and adds the fresh context to the existing
local continuation. The returned URL may be the same workspace URL because
the local thread is the same.

## 11. Full Alpha to Beta to Alpha round trip

The user starts in Alpha, moves to Beta, and later returns to Alpha.

```mermaid
sequenceDiagram
    actor User
    participant Alpha
    participant Beta

    User->>Alpha: Start task
    Alpha->>Beta: T + K-alpha-1 + Alpha context
    Beta-->>User: Beta continuation URL
    User->>Beta: Continue task
    Beta->>Alpha: T + K-beta-1 + Beta context
    Alpha-->>User: Alpha continuation URL
```

The wire values are:

| Transfer | Thread ID | Idempotency key | Client ID | Client origin | Destination |
| --- | --- | --- | --- | --- | --- |
| Alpha → Beta | `b66d4ea0-77d9-45d7-a58f-8c04dcb95692` | `7d1e3ef1-68c3-42f8-96a4-7d9a2f741252` | `alpha-agent` | `https://alpha.example` | Beta's discovered v1 endpoint |
| Beta → Alpha | `b66d4ea0-77d9-45d7-a58f-8c04dcb95692` | `234910ad-2c01-42a4-b3fb-b5d54fef6604` | `beta-agent` | `https://beta.example` | Alpha's discovered v1 endpoint |

Beta must return through the Alpha deployment from which the thread came.
Alpha maps both its original sent transfer and the returned received transfer
to the same local task. The idempotency keys differ because these are two
genuine transfers.

If Alpha later sends the task to Beta again, it keeps the thread ID and uses a
third new idempotency key.

## 12. Filtering private conversation state

Assume Alpha's internal history contains:

1. a system prompt with private policy text;
2. a user message;
3. hidden model reasoning;
4. a tool call containing an access token and internal index IDs;
5. a visible assistant answer;
6. an internal "handoff received" bookkeeping message; and
7. the user's latest request.

Alpha does not serialize the local history blindly. A safe handoff sample is:

```json
[
  {
    "role": "user",
    "content": [
      {
        "type": "text",
        "text": "Compare the termination rights in the selected agreements."
      }
    ]
  },
  {
    "role": "assistant",
    "content": [
      {
        "type": "text",
        "text": "The Northwind agreement appears broadest, but its change-of-control limitation still needs review."
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "text",
        "text": "Please verify that limitation in Beta."
      }
    ]
  }
]
```

Alpha omits the system prompt, hidden reasoning, secret tool call, and protocol
bookkeeping. If the tool found a document that Beta needs, Alpha attaches the
user-approved document as a resource instead of leaking the tool invocation.

## 13. Receiver pseudocode

This sketch emphasizes externally visible invariants rather than a particular
framework or database:

```python
def receive_handoff(http_request):
    principal = authenticate(http_request)
    target = authorize_target(principal, http_request.url)

    client = bind_client_identity(
        principal=principal,
        client_id=http_request.header("Handoff-Client-Id"),
        client_origin=parse_https_origin(
            http_request.header("Handoff-Client-Origin")
        ),
        target=target,
    )

    thread_id = validate_thread_id(
        http_request.header("Handoff-Thread-Id")
    )
    key = parse_uuid(http_request.header("Idempotency-Key"))
    package = validate_ahp_v1_json(http_request.json)

    scope = (target.tenant, target.user, client.identity, client.origin, key)
    fingerprint = hash_semantic_request(
        version="v1",
        target=target,
        client=client,
        thread_id=thread_id,
        package=package,
    )

    # A same-key request has one in-flight owner across processes.
    with idempotency_singleflight(scope):
        prior = load_idempotency_result(scope)
        if prior is not None:
            if prior.fingerprint != fingerprint:
                raise HttpConflict("idempotency key reused")
            return prior.response

        # Fetch into bounded temporary storage. Every URL and redirect is checked.
        resources = materialize_resources_atomically(
            package.context.resources,
            allowed_origins=client.resource_origins,
            credentials=client.resource_credentials,
        )

        with database_transaction():
            # Re-check inside the durable transaction if the lock implementation
            # does not itself guarantee exclusivity through commit.
            prior = load_idempotency_result_for_update(scope)
            if prior is not None:
                discard_temporary_resources(resources)
                if prior.fingerprint != fingerprint:
                    raise HttpConflict("idempotency key reused")
                return prior.response

            thread = find_or_create_thread(
                tenant=target.tenant,
                user=target.user,
                remote_client=client.identity,
                remote_origin=client.origin,
                thread_id=thread_id,
            )
            attach_handoff(
                thread=thread,
                objective=package.context.objective,
                conversation=package.context.conversation,
                resources=resources,
            )
            redirect = build_trusted_continuation_url(thread)
            response = {"redirect_url": redirect}
            store_idempotency_result(scope, fingerprint, response)

        schedule_agent_work(thread)
        return response
```

Production code also needs cleanup for abandoned temporary resources, bounded
locks, rate limiting, audit events, safe metrics, and careful handling of a
crash between commit and agent scheduling.

## 14. Sender pseudocode

The sender persists its pending transfer before the network call so a process
restart does not accidentally generate a second key:

```python
def handoff_to_receiver(local_thread, selected_context, receiver):
    review = build_minimized_package(selected_context)
    require_user_confirmation(receiver.display_name, review)

    thread_id = local_thread.ahp_thread_id or new_uuid()
    key = new_uuid()
    endpoint = discover_v1_endpoint(
        origin=receiver.discovery_origin,
        credentials=receiver.credentials,
    )

    request = PendingTransfer(
        endpoint=endpoint,
        target_identity=receiver.target_identity,
        thread_id=thread_id,
        idempotency_key=key,
        client_id=LOCAL_CLIENT_ID,
        client_origin=LOCAL_DEPLOYMENT_ORIGIN,
        body={"context": review},
    )
    persist_pending_transfer(request)

    while retry_policy_allows(request):
        try:
            response = post_unchanged_request(request)
            redirect = validate_trusted_redirect(
                response["redirect_url"], receiver.redirect_origins
            )
            mark_transfer_succeeded(request, redirect)
            local_thread.ahp_thread_id = thread_id
            return redirect
        except UnknownOutcomeOrRetryableFailure as error:
            wait_with_backoff_and_jitter(error.retry_after)

    raise HandoffUnavailable()
```

The loop reuses the stored request, including the original key. A validation
error corrected by changing the payload becomes a new pending transfer with a
new idempotency key.

## 15. Validation and error examples

### Missing resource content

This resource is invalid because both `blob` and `text` are absent:

```json
{
  "name": "agreement.pdf",
  "description": "The agreement to review."
}
```

Beta rejects the entire request with `422 Unprocessable Content`.

### Invalid base64

This is invalid because the value contains characters outside standard
base64:

```json
{
  "name": "agreement.pdf",
  "blob": {
    "type": "inline_blob",
    "mime_type": "application/pdf",
    "data": "not base64!"
  }
}
```

Beta validates all inline representations before materializing any resource
and rejects the whole transfer.

### Unconfigured client origin

If valid credentials for `alpha-agent` assert:

```http
Handoff-Client-Origin: https://attacker.example
```

Beta rejects the request. Matching an allowed client ID is not enough; the
authenticated identity, client ID, origin, target user, and tenant must agree.

### Unsafe redirect response

Alpha rejects this otherwise well-formed success response because the scheme
is not HTTPS and the destination is not trusted:

```json
{
  "redirect_url": "javascript:stealCredentials()"
}
```

Alpha also rejects an HTTPS redirect to an origin that was not configured for
Beta, unless that origin was explicitly allowlisted out of band.

## 16. Practical test matrix

Before enabling a partner integration, run both sides through this small
matrix:

| Scenario | Expected result |
| --- | --- |
| Valid minimal package | One continuation and `200` with a trusted URL. |
| Same key and same request | Same URL, no duplicate side effects. |
| Two concurrent identical requests | One continuation; both callers receive the same result. |
| Same key and changed objective | `409`, no changed state. |
| Same thread and new key | Existing scoped task resumes. |
| Same thread string from another user | Separate task; no information leak. |
| Inline text | Text is available to the receiving task. |
| Inline blob with invalid base64 | Whole transfer rejected. |
| Remote URL redirects to a private address | Whole transfer rejected. |
| Remote response exceeds limit | Whole transfer rejected without unbounded buffering. |
| Unknown client origin | Rejected before resource fetching. |
| Expired credential | `401` with no continuation. |
| Unsupported MCP block | `422` before continuation creation. |
| Receiver returns untrusted redirect | Sender rejects navigation. |
| Alpha → Beta → Alpha | Same thread resumes on both sides, with a new key per direction. |

These tests capture the protocol's central promise: one user-authorized
transfer has one durable effect, and a continuing task stays correlated
without sharing either product's internal conversation identifiers.
