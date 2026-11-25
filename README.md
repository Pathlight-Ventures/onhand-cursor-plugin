# Onhand for Cursor

MCP server for local shop inventory, starting in Dallas. Check stock from Square, Shopify POS, or a CSV. Search is read-only; sign-in is required for bookings and handoffs.

Onhand is operated by Pathlight Ventures in Texas. More on askonhand.com: [guides](https://askonhand.com/guides).

## What this repo contains

Configuration and documentation only.

- `.mcp.json`: points Cursor at the hosted Onhand MCP server
- `.cursor-plugin/plugin.json`: Cursor plugin manifest
- `README.md`, `SECURITY.md`, `LICENSE`

There is no code in this repo. Nothing runs on your machine, and there are no install scripts or hooks.
The server is operated by Onhand at `mcp.askonhand.com`; its source is not part of this repo.

## Install in Cursor

Add the server in **Settings → MCP**, or put this in `.cursor/mcp.json` (it is the same content as this repo's `.mcp.json`):

```json
{
  "mcpServers": {
    "onhand": {
      "url": "https://mcp.askonhand.com/mcp/cursor"
    }
  }
}
```

- Transport: Streamable HTTP.
- No headers, API keys or tokens are configured. Search works without signing in. Tools that act on your behalf
  prompt you to sign in (see [Authentication](#authentication)).
- MCP uses POST and OPTIONS. A plain `GET` to the URL returns 405; that is expected.
- The client surface (Cursor) is taken from the URL path, not from a tool argument.

Setup page: https://askonhand.com/use

## Tools

37 tools, as returned by `tools/list` on the live Cursor endpoint. Check the live list for the current set.

"Confirm step" means the write is two-step: a `prepare_*` (or `update_*`) tool stages an intent and returns the
consequence, then a matching `confirm_*` tool applies it. Per the tool descriptions, the frozen intent is valid
for ten minutes, and the assistant is told to get your approval of the exact consequence before confirming.
Prepare calls stage an intent only; they change nothing upstream.

### No sign-in: public search and lookup

| Tool | What it does | Mode |
| --- | --- | --- |
| `discover_local` | Search items, services and directory businesses by coordinates and optional city | Read-only |
| `search_local_offerings` | Search items and services by coordinates and optional city | Read-only |
| `find_business` | Find a business by name, optionally narrowed by town or street | Read-only |
| `find_nearby_businesses` | Directory search by category, coordinates and optional city (up to 25 results) | Read-only |
| `get_business` | Read a published business: hours, address, phone, website, FAQs, handoffs and more | Read-only |
| `ask_business` | Look up a question in a business's approved FAQs and owner-approved facts (retrieved, not composed) | Read-only |
| `get_open_slots` | Read up to 20 upcoming appointment windows for a business | Read-only |
| `check_availability` | Read one offering's stored availability; withheld when not eligible or stale | Read-only |
| `list_verified_reviews` | Read up to 20 receipt-backed reviews of a published business | Read-only |
| `forward_question` | Queue a question the business's FAQs could not answer. Needs your explicit consent | Writes |
| `request_area` | Record interest in a shop category near a place. Needs your consent; coordinates become a coarse area | Writes |
| `report_business_facts` | Submit up to five public business facts for verification. Needs your consent; never published by this call | Writes |

### Signed-in shopper actions

| Tool | What it does | Mode | Confirm step |
| --- | --- | --- | --- |
| `prepare_booking` | Stage a booking or provider handoff | Writes an intent only | Yes |
| `confirm_booking` | Create an eligible Square appointment, or record a URL handoff | Writes | Yes |
| `prepare_booking_cancellation` | Check policy and stage cancelling an Onhand-created Square appointment | Writes an intent only | Yes |
| `confirm_booking_cancellation` | Cancel the prepared appointment | Writes | Yes |
| `prepare_checkout` | Stage checkout for one eligible item | Writes an intent only | Yes |
| `confirm_checkout_handoff` | Create an eligible hosted checkout, or record a URL handoff. You pay on the provider's site | Writes | Yes |
| `prepare_quote_request` | Stage a handoff to a verified quote URL. No contact details are sent | Writes an intent only | Yes |
| `confirm_quote_handoff` | Record the quote handoff. You submit your details on the provider's site | Writes | Yes |
| `get_action_status` | Read the status of one of your actions | Read-only | No |
| `report_action_outcome` | Record that a handoff completed, failed or was abandoned (once per action, within 72 hours) | Writes | No |
| `submit_verified_review` | Publish a rating and review for a provider-verified completed checkout. Needs your approval of the exact text | Writes | No (approval of the text) |
| `list_subjects` | List organizations for your sign-in | Read-only | No |
| `select_subject` | Choose which organization later merchant tools use | Writes a selection | No |
| `setup_account` | Save a shopper account with an optional display name, or an intent to claim a business | Writes | No |
| `start_business_claim` | Return a claim link for a business. Ownership is proven on the linked page, not by this call | Writes | No |

A handoff is not a completed booking or purchase. Onhand does not take payment.

### Merchant tools (business owner or team)

Sign-in and an active organization membership are required. Permissions come from your role; see
[Authentication](#authentication).

| Tool | What it does | Mode | Confirm step |
| --- | --- | --- | --- |
| `get_company` | Read your company profile | Read-only | No |
| `list_locations` | List your locations | Read-only | No |
| `list_connections` | List connected systems and sync health; no credentials | Read-only | No |
| `get_subscription` | Read your subscription status; no Stripe identifiers | Read-only | No |
| `update_company_profile` | Stage a change to name, website, phone or categories | Writes an intent only | Yes |
| `confirm_update_company_profile` | Apply the staged profile change | Writes | Yes |
| `update_hours` | Stage opening hours for a location | Writes an intent only | Yes |
| `confirm_update_hours` | Apply the staged hours | Writes | Yes |
| `get_connection_link` | Return a dashboard link to connect, reconnect, map or disconnect a source; the change happens in the dashboard | Read-only | No |
| `get_billing_link` | Return a dashboard link for billing; changes happen there | Read-only | No |

Merchant reads return only your own organization's data.

## Authentication

Public search and lookup work without sign-in. Protected tools use OAuth 2.1 authorization code flow with PKCE
(S256), through WorkOS AuthKit. Cursor handles the flow.

- The access token is sent as `Authorization: Bearer <token>`.
- Onhand validates the issuer, audience, signature, expiry and active-token status.
- Protected writes fail closed when the issuer cannot be reached.
- A 401 response carries a `WWW-Authenticate` challenge with `resource_metadata`, pointing at the
  [protected-resource document](https://mcp.askonhand.com/.well-known/oauth-protected-resource/mcp/cursor).
  That document lists the authorization server and the supported scopes: `openid profile email offline_access`.
- Onhand does not register OAuth clients itself; client registration is handled by AuthKit.
- Tool permissions (for example `actions:confirm`, `company:write`) are derived from your identity. They are not
  OAuth scopes a client requests.
- Merchant permissions depend on organization membership and role. With several organizations, call
  `list_subjects`, then `select_subject`.

Details: [docs, Authentication](https://askonhand.com/docs).

## Data handling

This section summarizes the [privacy policy](https://askonhand.com/privacy) (last updated November 11, 2026).
The policy is the authoritative text; read it for the full picture.

**What the server receives**

- Tool arguments, such as a search term, coordinates or a city.
- If you sign in to take an action: an account identifier and your verified email address, from the sign-in providers.
- Searches and business lookups work without signing in.

**Stated in the policy**

- Onhand never stores exact stock counts, never takes payments, and never stores payment card details.
- It does not sell personal information, show ads, or let anyone pay to rank higher.
- Your details are shared with a business or its provider only when you confirm an action, and you are told
  what will be shared first. For example, a Square appointment receives your verified email.
- The assistant you use receives the results of the tools it calls.
- Rate limiting: IP addresses, and for MCP requests the session identifier when supplied, are stored unhashed in
  short-term rate-limit counters. Expired counters are cleared as new requests arrive and otherwise by the team,
  and may remain until then.
- Empty-search records: when you are signed in and a search finds no matching item at a nearby shop, a record with
  your search text and the shop may be kept so nearby shops can see demand. It carries no account identifier and has
  no fixed deletion schedule. It is removed if the shop's listing is deleted. Avoid typing personal information
  into searches.
- Forwarded questions: nothing is forwarded without your explicit agreement to the exact text and business. The
  business receives the question, category and city, and your first name and reply email only if you opt in. A
  one-way hash of the caller's network address (not the address) is kept with the business ID for up to 24 hours
  to enforce daily limits.
- Sensitive fields are kept out of logs.
- Service providers named in the policy: Google Cloud (hosting), PlanetScale (database), Google Firebase
  Authentication and WorkOS (sign-in), Stripe (merchant billing), Resend (email), and an AI model provider that
  reads public business web pages, menu PDFs and images.

**Retention**

The policy frames retention periods as targets: "Our team currently deletes records manually and periodically.
Deletion is not automated. Records may remain past the target until removed." Periods it states include:

- Short-lived records such as pending action requests: generally up to about 90 days, longer while an action is
  still being completed or reconciled.
- Many operational records: up to about 180 days.
- Records needed to settle a later dispute (ownership claims, listing decisions, audit logs, history of shopper
  actions you approve): about two years, and possibly longer because deletion is manual.

Other categories (website checks, analytics, merchant data) have their own entries in the policy.

**Your choices**

Email privacy@askonhand.com to access, correct or delete your information. The policy says they reply within 45
days. You can disconnect Onhand from your assistant at any time in the assistant's settings.

This repo does not describe or promise anything beyond what the policy and docs say. Anything this README does
not state, such as log contents or a specific retention period for a given record, is not published here.

## Links

| | |
| --- | --- |
| Privacy policy | https://askonhand.com/privacy |
| Terms | https://askonhand.com/terms |
| Docs | https://askonhand.com/docs |
| Setup | https://askonhand.com/use |
| Support | help@askonhand.com |
| Privacy requests | privacy@askonhand.com |
| Security reports | see [SECURITY.md](SECURITY.md) |

## License

[MIT](LICENSE). The license covers the files in this repo, not the hosted service, which is governed by the
[Terms](https://askonhand.com/terms).
