# Lish

Lish is a corporate catering and workplace food service company serving Seattle
and the Bellevue/Eastside area of Washington, including Redmond and Kirkland. It
connects employers with a curated roster of local partner chefs and runs the whole
workplace meal program — subscription catering, PopUp restaurants, on-demand
catering, and occasion catering for breakfast, lunch, box lunches, happy hours,
meetings and events.

Backed by: 500 Global — https://www.lishfood.com

## API status

**Lish operates no developer program and publishes no product API.** There is no
developer portal, no API documentation, no SDK, no CLI, no MCP server, no webhook
or event surface, no status page and no changelog. Probed 2026-07-19:
`api.lishfood.com`, `developers.lishfood.com` and `status.lishfood.com` do not
resolve, and every `/.well-known/` path returns 404.

The one genuinely public, self-describing HTTP API is the **WordPress REST API**
that backs the marketing site and blog, at `https://www.lishfood.com/wp-json/`.
It exists as a byproduct of the CMS. Anonymous reads need no credentials. It is
useful for reading Lish content and nothing else — there is no published
versioning, deprecation, SLA or rate-limit policy, so it carries no stability
contract.

There is no public API for menus, pricing, availability, ordering, delivery status
or accounts. Those live behind a login in the customer application at
`https://app.lishfood.com`.

## Artifacts

| Path | Type | Method |
|---|---|---|
| `apis.yml` | APIs.json profile | — |
| `openapi/lish-wordpress-openapi.json` | OpenAPI 3.1, 18 operations | derived from the live route index |
| `overlays/lish-wordpress-overlay.yaml` | OpenAPI Overlay 1.0.0 | generated |
| `conventions/lish-conventions.yml` | Conventions | derived, verified live |
| `errors/lish-problem-types.yml` | ErrorCatalog | derived, all codes verified live |
| `lifecycle/lish-lifecycle.yml` | Lifecycle | derived |
| `authentication/lish-authentication.yml` | Authentication | derived |
| `conformance/lish-conformance.yml` | Conformance | derived |
| `data-model/lish-data-model.yml` | DataModel | derived |
| `security/lish-domain-security.yml` | DomainSecurity | probed |
| `well-known/lish-well-known.yml` | probe record (all 404) | probed |
| `llms/lish-llms.txt` | LLMsTxt | generated |
| `mcp/lish-mcp.yml` | candidate MCP surface — nothing published by Lish | derived |
| `skills/` | 2 Agent Skills + index | generated |

Not applicable for this provider: packages/SDKs, gRPC, AsyncAPI/webhooks, OAuth
scopes, sandbox, CLI, components, changelog, decline codes, vulnerability
disclosure, trust center.

Live content counts as of 2026-07-19: 28 blog posts, 60 pages, 4 categories.
