# OpenOrange Migration Plan (Live CF OS)

Generated from live Cloudflare account `jo` (`fe4630ea4d2289fa66ce3e2752fff191`) on 2026-11-23.

## Live deployed workers (source of truth)

| Worker | Compat date | Flags | Bindings | DO classes (namespaces) |
|---|---|---|---|---|
| `jo-os-router` | 2026-09-04 | - | ASSETS; service: GATEKEEPER_CONTEXT→jo-os-context, GATEKEEPER_CUSTOMGATEKEEPER→jo-os-custom-gatekeeper, GATEKEEPER_SCHEDULER→jo-os-scheduler, GATEKEEPER_WEBSEARCH→jo-os-websearch, WORKSHOP_BACKEND→workshop-backend | none |
| `workshop-backend` | 2026-09-04 | allow_irrevocable_stub_storage, global_fetch_strictly_public, nodejs_compat | D1 AUDIT_DB(fdf9a555-8d0f-4d39-958f-3ee0ef73b0d9); KV AVATARS(188f4343...), BLUEPRINT_CONTENT(b8f08ea1...), BLUEPRINTS(f3346e56...); Browser BROWSER; version_metadata CF_VERSION_METADATA; DO GADGETS→GadgetServer(7810378e...); services to 4 gatekeepers | AdminSettings, GadgetServer, UserDurableObject, PendingLogin, AdminAuditDurableObject, UserDirectoryDurableObject, OverseerDurableObject |
| `jo-os-context` | 2026-09-04 | nodejs_compat, allow_irrevocable_stub_storage | KV CONTEXT_COLLECTIONS(c26a9ed0...) | LibraryRegistryDurableObject, ContextCollectionDurableObject, UserLibraryDurableObject, ContextGatekeeper |
| `jo-os-custom-gatekeeper` | 2026-09-04 | allow_irrevocable_stub_storage, nodejs_compat | plain_text CUSTOM_MESSAGE, CUSTOM_NAME | CustomGatekeeper |
| `jo-os-scheduler` | 2026-09-04 | allow_irrevocable_stub_storage, nodejs_als | none | ScheduleDriver, SchedulerGatekeeper |
| `jo-os-websearch` | 2026-09-04 | allow_irrevocable_stub_storage | AI binding AI; plain_text CF_AI_GATEWAY | WebSearchGatekeeper |

## DO inventory snapshot (CF API, limit 10)

- workshop-backend_OverseerDO: 10+ objects, stored data present
- workshop-backend_GadgetServer: 2+ objects
- workshop-backend_AdminSettings/UserDO/UserDirectoryDO: 1+ objects, stored data present
- All other namespaces: 0 objects seen at snapshot time

## Known CF limits / why migrate

- Verify current values in CF dashboard/docs: [DO pricing](https://developers.cloudflare.com/durable-objects/platform/pricing/), [DO limits](https://developers.cloudflare.com/durable-objects/platform/limits/).
- OpenOrange/Rivet target removes CF DO platform caps (storage/CPU/wall/connections/bundle/nodejs_compat subset) by running DOs as Rivet Actors on stock Node.js. Validate with staging before cutover.

## Keep `jo-os-router.nosleep.workers.dev` working

**Hybrid (recommended):** keep `jo-os-router` on Cloudflare as public entrypoint; deploy DO-owning backend workers to OpenOrange on Rivet; proxy/service-bind from CF router to Rivet pool URL where possible. This preserves the workers.dev URL while moving DO pressure off CF.

**Full cutover:** deploy all 6 workers to Rivet; move custom domain to Rivet pool URL. `workers.dev` remains CF-hosted only if you keep a CF worker there.

## Migration blockers / mapping

| Binding/capability | Current | OpenOrange action |
|---|---|---|
| Durable Objects (sqlite) | 15 namespaces | `orange migrate` / export-import; validate class exports/migrations |
| Service bindings | router↔backend↔gatekeepers | OpenOrange multi-worker/service bindings are partial; validate staging topology |
| D1 AUDIT_DB | workshop-backend | `orange d1 migrations apply` / D1 migration flow |
| KV (AVATARS, BLUEPRINTS, BLUEPRINT_CONTENT, CONTEXT_COLLECTIONS) | workshop-backend, jo-os-context | Explicit export/import scripts |
| Browser Rendering | workshop-backend BROWSER | Pluggable provider |
| Workers AI | jo-os-websearch AI | Replace with pluggable/OpenAI-compatible provider; map CF_AI_GATEWAY |
| Assets | jo-os-router ASSETS | Validate asset serving/routing |
| version_metadata | workshop-backend CF_VERSION_METADATA | Verify/shim |
| Secrets/env | plain_text + secrets in CF | Recreate via `--secrets-file`/`--var` and Rivet secrets; never commit |

## Deploy target (Rivet)

- Org: `jomon-solinaps-organization-gxoi`
- Cluster: `curl-trench` (curltrench5f4z-api.onrivet.dev)
- Runner pools/builds: none currently provisioned
- Deploy requires Docker-capable runner (Linux/macOS/WSL2+Docker or GitHub Actions).

## CI/CD

See `.github/workflows/openorange-deploy.yml`. Requires `RIVET_CLOUD_TOKEN` secret (cluster token, Read+Deploy). First deploy may need manual pool creation/dashboard step depending on token scope.
