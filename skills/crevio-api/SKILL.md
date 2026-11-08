---
name: crevio-api
description: "Work with a Crevio business: products, checkouts, orders, customers, email, sites, socials, ads and the rest of the API, through the Crevio MCP server (delegate to the Crevio agent or run scripts), the @crevio/sdk TypeScript client, or REST."
---

# Crevio API

Crevio is an AI platform that runs a small business. The API covers the whole business: products and prices, checkouts, orders, refunds, subscriptions, invoices, discounts, customers and tags, forms, bookings, email (broadcasts, sequences and mailboxes), websites, domains, social media, paid ads, phone calls and texts, image, video and audio generation, web research, AI tasks, webhooks, and usage.

Every resource belongs to one account (the business). A credential only ever sees its own account.

## Choose a way in

| You are | Use |
|---|---|
| An agent connected to the Crevio MCP server | Hand work to the Crevio agent, or run scripts against the API. See [MCP server](#mcp-server) |
| Writing TypeScript | `@crevio/sdk`. See [SDK](#typescript-sdk) |
| Anything else | REST at `https://api.crevio.co/v1`. See [REST](#rest) |

New here? `https://api.crevio.co/register.md` walks through getting a key. `GET https://api.crevio.co/health` (no auth) checks liveness, and `GET /v1/status` confirms which account a key resolves to.

## MCP server

```
URL:        https://mcp.crevio.co/mcp   (Streamable HTTP)
Discovery:  https://mcp.crevio.co/mcp/server-card
```

Sign in with OAuth 2.1 (protected resource metadata at `https://mcp.crevio.co/.well-known/oauth-protected-resource`, dynamic client registration at `https://crevio.co/oauth/register`), or send an API token as `Authorization: Bearer <token>`.

The person on the other end usually runs the business and is not a developer. Talk about their products, customers, orders and emails in plain words, and leave ids, tool names and API fields out of replies unless they ask.

The server does not expose one tool per endpoint. It has two surfaces:

| The job | Use | Typical time |
|---|---|---|
| Exact numbers, lookups, lists, bulk changes | `run_script` | Seconds |
| Writing, judgement, research, work across several apps | `ask_crevio` (waits) or `start_chat` (returns at once) | A minute or more |

`whoami` shows the account, plan, credit balance and limits.

### Delegating to the Crevio agent

- `ask_crevio` starts a job and waits up to 300 seconds for the agent's final reply. `start_chat` queues it and returns the run right away; follow it with `wait_for_run` or `get_run`.
- `send_message` continues a chat. The agent keeps its context.
- `list_runs`, `list_chats` and `list_messages` read past work, including scheduled tasks.
- `approval_mode: "supervised"` pauses the run in `needs_input` before each change. Show the person what is pending, then answer with `resolve_approvals`. `read_only` forbids changes.

### Rules for every job

- **Always pass an `idempotency_key`** to `ask_crevio`, `start_chat`, `send_message` and `run_script`. Derive it from the task (`weekly-review-2026-11-08`) so a retry returns the first run instead of doing the work, and spending credits, twice.
- **`wait_timed_out: true` is not an error.** The work is still running. Keep waiting with `wait_for_run` or `wait_for_script_run`. Never start it again.
- **Ask before anything reaches customers or money.** Sending email or texts, publishing, refunds, discounts, price changes and deletions need the person's clear yes to exactly what will happen.

### Scripts

`run_script` runs JavaScript (bun) on the account's Crevio computer, with no agent turn. Each namespace under `tools` is a set of functions:

- `crevioApi`: the whole REST API, one function per operation (`listOrders`, `createProduct`, ...).
- `platform`: Crevio's own tools.
- One namespace per connected app (`pdGmail`, `mcpLinear`, ...).

**Find the function first.** `list_integrations` names the namespaces. `list_integration_tools` lists a namespace's functions with their parameters as TypeScript types, whether each only reads, whether it needs approval, and a snippet. `crevioApi` has hundreds of operations, so always pass `search`:

```json
{ "namespace": "crevioApi", "search": "discount create" }
```

**Call shape.** Top-level `await` works, a failed call throws, and the script's last expression is its result.

- Reads take their parameters directly: `tools.crevioApi.listOrders({ status: "succeeded", limit: 100 })`.
- Writes put the request body under `body`, with path parameters beside it: `tools.crevioApi.updateProduct({ id_or_slug: "prod_abc", body: { status: "active" } })`.

```js
const product = await tools.crevioApi.createProduct({ body: { name: "My Course", status: "draft" } });
const price = await tools.crevioApi.createPriceVariant({
  body: { product: product.id, name: "Standard", amount_type: "fixed", amount: 4900, currency: "usd", billing_type: "one_time" },
});
await tools.crevioApi.updateProduct({ id_or_slug: product.id, body: { status: "active" } });
({ product: product.id, price: price.id })
```

**Read everything when the answer must be complete.** Lists return at most 100 items per page:

```js
const all = [];
let starting_after;
do {
  const page = await tools.crevioApi.listCustomers({ limit: 100, starting_after });
  all.push(...page.data);
  starting_after = page.data.at(-1)?.id;
  if (!page.has_more) break;
} while (starting_after);
all.length
```

**Approvals.** A function marked `requires_approval: true` does not run. The call resolves to `{ pending: true, draft_id, approval }` and the script carries on, so check `result.pending` before you use a write's result. The run lists it in `pending_approval_ids`. Approving it (in Crevio, or with `resolve_approvals` once the person confirms) runs that one frozen call, never the whole script again. Which functions need approval depends on the account's settings, so read the listing rather than assuming.

**Inputs and files.** Pass values with `args` (up to 16 strings) and read them from the global `args`; pass long text this way rather than pasting it into the code. Files written into the directory in the global `outputDir` come back as `artifacts` (20 at most, 50 MB each); `get_file_download_url` turns one into a link.

**Return small results.** Return a summary object, not whole records. The reply is read by a model.

**Limits.** Scripts run as the signed-in person and only for account admins; if `run_script` is refused, use `ask_crevio` instead. A script may run up to 600 seconds (default 300). At most 4 runs at a time per person. Only a queued run can be cancelled.

## TypeScript SDK

```bash
npm add @crevio/sdk
# or: bun add @crevio/sdk / pnpm add @crevio/sdk
```

Works in Node.js 20.19+, Bun, Deno, Cloudflare Workers and browsers. Methods mirror the API reference (`crevio.products`, `crevio.email.inboxes.messages`, `crevio.sites`, `crevio.ads`, ...). Fields are camelCase in TypeScript and sent as snake_case.

```typescript
import { Crevio } from "@crevio/sdk";

const crevio = new Crevio({ apiKey: process.env["CREVIO_API_KEY"] ?? "" });

const products = await crevio.products.list({ limit: 10 });

const product = await crevio.products.create({ name: "My Course" });

const checkout = await crevio.checkouts.create({
  lineItems: [{ priceVariant: "price_abc123", quantity: 1 }],
});
// Send the buyer to checkout.purchaseUrl.
```

- Failed requests throw. API errors carry the status code and the parsed error.
- An action that needs approval returns an object with `object: "approval"` instead of running. Check it, then read `crevio.approvals.get({ id })` later; an approval ends `executed`, `failed` or `denied`.
- Requests are not retried unless you pass a `retryConfig`.

## REST

```
Base URL:  https://api.crevio.co/v1
Auth:      Authorization: Bearer <api_token>
Spec:      https://api.crevio.co/openapi.json
Reference: https://crevio.co/docs/developer/api-reference/introduction
```

Create tokens at `https://crevio.co/app/settings/developer`. A token acts for the person who created it, on one account.

### Conventions

- **Bodies are flat.** Send fields at the top level: `{"name": "My Course"}`, not `{"product": {"name": "My Course"}}`. A wrapped body is rejected with `400 parameter_unknown`.
- **Keys** are `snake_case`.
- **Ids** carry a type prefix: `prod_`, `price_` (price variants), `co_` (checkouts), `cus_`, `ord_`, `inv_`, and so on. References use the bare name with an id: `"product": "prod_abc"`, not `product_id`.
- **Money** is an integer in the smallest currency unit: `4900` is $49.00.
- **Lists** return `{ "object": "list", "data": [...], "has_more": bool, "url": "..." }`. Page with `limit` (at most 100, default 20) and `starting_after=<last id>` until `has_more` is false.
- **Expansion:** `expand=customer,line_items` (comma-separated) returns those relations as full objects instead of ids.
- **Idempotency:** send `Idempotency-Key: <uuid>` on `POST` to retry safely.
- **Rate limits** depend on the plan: 600, 1,800 or 3,000 requests a minute per credential, lower on a few costly routes. Responses carry `RateLimit` and `RateLimit-Policy`; a `429` adds `Retry-After`.
- **Versioning:** the version is in the path. Additive changes ship to `v1`, removals never do. A deprecated operation sends `Deprecation`, `Sunset` and a `Link` header for at least six months first. Policy: `https://crevio.co/docs/developer/guides/versioning`.
- **Errors:**
  ```json
  { "error": { "type": "invalid_request_error", "code": "parameter_missing", "message": "...", "param": "name" } }
  ```
  `401` missing or expired token, `403` missing scope, `404 resource_missing` wrong id, `400 parameter_missing` or `parameter_unknown` bad body, `422` validation failed. Cite the `x-request-id` response header when asking for help.
- **Approvals:** an action the account has set to ask first answers `202 Accepted` with an approval object instead of running.

### How the business fits together

- A **product** is what is for sale. It can't go `active` until it has a **price variant** (`amount_type`: `free`, `fixed` or `custom`; `billing_type`: `one_time`, `subscription` or `payment_plan`).
- What the buyer gets is an **experience**: `course`, `digital_download`, `written_content`, `forum`, `discord_server`, `telegram`, `event`, `event_type` (bookable session), `link` or `website_embed`. Link it to a price variant with `POST /price_variants/{id}/experiences`; buying that variant grants it.
- A **checkout** (`POST /checkouts`) or a shareable **checkout link** (`POST /checkout_links`) sells price variants. Send the buyer to the checkout's `purchase_url`; the order arrives through the `order.paid` webhook.
- An **order** belongs to a **customer** (`customer_type`: `lead`, `free` or `paid`). Customers carry tags and `metadata`.
- A **broadcast** is one email to many people (specific addresses, a customer type, or everyone). It is saved as a draft; preview it, send a test, then send. A **sequence** is an automated series of emails. **Mailboxes** under `/email/inboxes` hold one-to-one threads.

### Where things live

Exact operations, parameters and schemas are in the OpenAPI spec.

| Area | Paths |
|---|---|
| Account and team | `/account`, `/me`, `/team`, `/usage`, `/plans`, `/auth/keys` |
| Overview | `/analytics` (revenue, orders, leads, customers, refunds, top products and traffic for a date range, against the previous period) |
| Catalog | `/products`, `/price_variants`, `/experiences` (plus chapters, lessons, files, topics, posts, sessions, grants), `/discounts`, `/reviews`, `/tags` |
| Selling | `/checkouts`, `/checkout_links`, `/checkout_configuration`, `/orders`, `/order_items`, `/refunds`, `/subscriptions`, `/invoices`, `/billing_portal/sessions` |
| Customers and forms | `/customers`, `/forms`, `/form_submissions`, `/waitlist_entries` |
| Bookings | `/event_types`, `/bookings`, `/schedules` |
| Email | `/broadcasts`, `/sequences`, `/email/inboxes` (threads, messages, drafts), `/email/suppressions`, `/email/metrics` |
| Phone and texts | `/calls`, `/phone-numbers`, `/sms-messages`, `/sms-follow-ups`, `/phone-consents`, `/phone-suppressions` |
| Websites and content | `/sites` (deployments, analytics, secrets, database), `/pages`, `/blog_posts`, `/legal_pages`, `/files` |
| Domains | `/domains` (search, check, purchase, verify), `/zones/{id}/records` |
| Social media | `/socials` (accounts, posts, comments, queue, uploads) |
| Paid ads | `/ads` (connect, campaigns, ad groups, audiences, lead forms, pixels, reports, insights) |
| Media generation | `/images`, `/videos`, `/audio`, polled through `/jobs` |
| Web research | `/web/search`, `/web/read`, `/web/map`, `/web/crawl`, `/web/extract`, `/web/research` |
| AI agent | `/chats`, `/tasks`, `/task_runs`, `/approvals`, `/script_runs`, `/skills`, `/bots` |
| Connected apps | `/connections` (connect, list tools, execute, proxy) |
| Events and webhooks | `/events`, `/event_sources`, `/webhook_endpoints`, `/webhook_events` |
| Business formation | `/formations` |

Files upload in two steps: `POST /files` returns a presigned URL, `PUT` the bytes there, then `POST /files/{id}/uploaded`.

## Webhooks

Create an endpoint with `POST /webhook_endpoints`; the response carries the `whsec_` signing secret, shown only once. Event types: `GET /webhook_endpoints/events`.

- **Verify with a library, never hand-written HMAC.** Deliveries follow [Standard Webhooks](https://www.standardwebhooks.com). In JavaScript: `npm install standardwebhooks`, then `new Webhook(secret).verify(rawBody, headers)`, which throws on a bad or stale signature. Pass the whole `whsec_…` string and the **raw** body (`await request.text()`), never re-serialized JSON.
- **Payload:** `{id, object: "event", type, api_version, created_at, data: {object: <the resource as its GET endpoint returns it>}}`.
- **Deduplicate on `id`** (same as the `webhook-id` header, identical on every retry). Delivery is at least once and unordered.
- **Answer `2xx` within 5 seconds**, then do slow work in the background. Failures retry for about three days; an endpoint that fails everything for 72 hours is switched to `inactive`.
- **Versioning:** an endpoint stays on the `api_version` it was created on; opt in to a newer one with `PATCH /webhook_endpoints/{id}`.
- **Test a handler:** `POST /webhook_endpoints/{id}/test?event_type=<subscribed type>` sends a signed test event and returns the receiver's `response_code` and `response_body`.
- **Deliveries:** `GET /webhook_events` (`expand=request` for the response); resend with `POST /webhook_events/{id}/redeliver`.

## Support

- Developer docs: `https://crevio.co/docs/developer`
- Support: `support@crevio.co`
