---
name: crevio-api
description: "Work with a Crevio business: products, checkouts, orders, contacts, email, sites, socials, ads and the rest of the API, through the Crevio MCP server (delegate to the Crevio agent or run scripts), the @crevio/sdk TypeScript client, or REST."
---

# Crevio API

Crevio is an AI platform that runs a small business: products and prices, checkouts, orders, subscriptions, discounts, contacts, bookings, email, websites, social media, ads, phone calls and texts, and more. Every resource belongs to one account (the business), and a credential only ever sees its own account.

This skill holds the rules to follow every time. Everything else is in Crevio's documentation; see [Finding the details](#finding-the-details).

## Ways in

| You are | Use |
|---|---|
| An agent connected to the Crevio MCP server (`https://mcp.crevio.co/mcp`, OAuth or `Authorization: Bearer <token>`) | `ask_crevio` / `start_chat` to hand a job to the Crevio agent, or `run_script` to call the API directly |
| Writing TypeScript | `@crevio/sdk` |
| Anything else | REST at `https://api.crevio.co/v1` with `Authorization: Bearer <api_token>` |

On the MCP server, use a script for exact numbers, lookups and bulk changes (seconds), and hand a job to the agent for writing, judgement or work across several apps (a minute or more). `list_integrations` and `list_integration_tools` (with `search`) show what a script can call; `tools.<namespace>.<function>(args)` makes a call, and the script's last expression is its result.

## Rules for every job

- **Talk to the merchant in plain words.** The person is usually a business owner, not a developer: leave ids, field names and API details out of replies unless they ask.
- **Ask before anything reaches customers or money.** Sending email or texts, publishing, refunds, discounts, price changes and deletions need a clear yes to exactly what will happen.
- **Make retries safe.** Pass an `idempotency_key` to `ask_crevio`, `start_chat`, `send_message` and `run_script`, and an `Idempotency-Key` header on REST `POST`s, so a retry returns the first result instead of doing the work twice.
- **`wait_timed_out: true` is not an error.** The work is still running: keep waiting with `wait_for_run` or `wait_for_script_run`, never start it again.
- **Approvals are a normal result.** An action the account has set to ask first doesn't run: a script call resolves to `{ pending: true, approval }`, REST answers `202` with an approval, and a supervised run pauses in `needs_input`. Say it is waiting for the person; don't report it as done.

## Conventions

- **Bodies are flat.** Send `{"name": "My Course"}`, not `{"product": {...}}`. In a script, a write's fields go under `body`, with path parameters beside it.
- **Money** is an integer in the smallest currency unit: `4900` is $49.00.
- **Ids** carry a type prefix (`prod_`, `price_`, `con_`, `ord_`), and references use the bare name: `product: "prod_abc"`, not `product_id`.
- **Lists** return `{ object: "list", data, has_more }`, at most 100 a page. Pass the last item's id as `starting_after` until `has_more` is false when the answer must be complete.
- **Operations** are named by their OpenAPI `operationId`: `tools.crevioApi.<operationId>` in a script, the matching SDK method, the operation of that name in the spec.

## How the business fits together

- A **product** is what is for sale. It can't go `active` until it has a **price variant**.
- What the buyer gets is an **experience** (course, download, community, event, bookable session, link), linked to a price variant so buying it grants access.
- A **checkout** or a shareable **checkout link** sells price variants; the order arrives through the `order.paid` webhook.
- An **order** belongs to a **contact**. A contact's `lifecycle` is `lead` or `customer`; Crevio moves a lead to customer when they first buy or get access, and it never moves back.
- A **broadcast** is one email to many people, a **sequence** is an automated series, and **inboxes** hold one-to-one threads.

## Finding the details

Look things up rather than guessing field names or behaviour:

- **Documentation:** on the MCP server, `search_docs` searches Crevio's help center and developer guides, and `read_docs_page` reads a page. Elsewhere, `https://crevio.co/docs/developer` and its index at `https://crevio.co/docs/llms.txt`.
- **Exact operations and fields:** the OpenAPI spec at `https://api.crevio.co/openapi.json`, or `list_integration_tools` on the MCP server.
- **Support:** `support@crevio.co`.
