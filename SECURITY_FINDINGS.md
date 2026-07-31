# Bifrost — Security Assessment Findings & Coordinated-Disclosure Plan

> **CONFIDENTIAL — contains working details of UNFIXED vulnerabilities.**
> Do not publish this file, open it in a public pull request, or push it to a
> public fork until the coordinated-disclosure steps in §3 have completed.
> If `matiasinsaurralde/bifrost` is a public fork, keep this on a private branch.

- **Target:** `matiasinsaurralde/bifrost` @ `2c668cd3b` (`dev`, V2.0.0), 2026‑07‑31
- **Assessment:** dependency-focused zero-day hunt + application authorization review
- **Method:** manual source analysis of Bifrost + its dependency/sub-dependency source in the Go module cache; each confirmed finding was reproduced and/or the exact code path was read end-to-end. Known-CVE scanning (govulncheck) was used only as a weak baseline — every finding below is an **undisclosed** issue.

---

## 1. Threat model

Bifrost has two request planes:

- **Management plane** `/api/*` — gated by `APIMiddleware` (admin basic-auth / bearer session / cookie / scoped temp-token). `allowVirtualKeyAuth=false`, so a virtual key never authorizes `/api/*`.
- **Inference plane** `/v1/*`, `/mcp`, `/v1/realtime/*` — `InferenceMiddleware` always passes through; authorization is delegated entirely to the governance plugin (virtual key).

Two deployment postures are referenced:

- **DEFAULT** — `EnforceAuthOnInference=false` (inference is unauthenticated by default) and no admin credentials configured (`IsLocalAdmin=true`, all `/api/*` open).
- **HARDENED** — `EnforceAuthOnInference=true` and admin credentials configured. **All §Confidentiality/Integrity findings below hold on a HARDENED instance.**

**Cross-cutting amplifier:** Bifrost runs on `fasthttp` v1.71.0 with **no `PanicHandler`** and only two `recover()` sites (`core/lib/ctx.go:203`, `handlers/websocket.go:165`). Any unrecovered panic — including in a background goroutine (WebRTC/DTLS/media pumps) or a `fatal error` such as a stack overflow — terminates the **entire process**, dropping all tenants.

---

## 2. Executive summary

Nine confirmed findings: five availability (remote DoS), four confidentiality/integrity (authorization/VK bypass). Two additional avenues (non-admin RCE, LFI) were investigated and came back **negative** — a positive assurance signal.

| ID | Finding | Class | Root cause owner | Reachability | Sev |
|----|---------|-------|------------------|--------------|-----|
| **A1** | `pion/webrtc` `getRids` simulcast SDP slice-panic | DoS (full crash) | **pion/webrtc** (Bifrost exposes) | unauth | High |
| **A2** | `pion/dtls` `fragmentBuffer.pop()` nil-deref | DoS (full crash) | **pion/dtls** (Bifrost exposes) | unauth | High |
| **A3** | `gjson` `ValidBytes` unbounded recursion (stack overflow) | DoS (full crash) | **gjson** + Bifrost misuse | unauth | High |
| **A4** | zstd window pre-allocation memory bomb | DoS (OOM) | **Bifrost** (dep misuse) | unauth | High |
| **A5** | WebSocket unbounded message (no `SetReadLimit`) | DoS (OOM) | **Bifrost** (dep misuse) | unauth | High |
| **C1** | Realtime proxy opens paid upstream with **no virtual key** | VK bypass / billing | **Bifrost** | unauth | High |
| **C2** | MCP tool-authz bypass via client-name dash-prefix confusion | Broken authz | **Bifrost** | 1 low-priv VK | Med-High |
| **C3** | MCP user-OAuth token for VK-less user fails open to global server | Broken authz | **Bifrost** | VK-less SSO user | High |
| **C4** | Admin-auth bypass via raw-vs-normalized path divergence | AuthN bypass | **Bifrost** (+ fasthttp/router footgun) | unauth | High* |

\* C4 shares its root cause with the previously-known `/api/skills/serve` bypass; it is a **generalization** of that bug (see the entry), wire-verified, with a practical-impact ceiling noted.

---

## 3. Disclosure strategy (the routing decision)

### 3.1 Who owns each finding

- **Bifrost owns all nine.** As the integrator it is the single choke point that can mitigate every finding in its own code, and it is the party with the concrete, deployed, reachable exposure.
- **A genuine dependency root cause exists for three findings only:** A1 (`pion/webrtc`), A2 (`pion/dtls`), and A3 (`gjson`, with a caveat). These are bugs in the dependency's own code, affecting **every** consumer of that library, not just Bifrost.
- **A4 and A5 are NOT dependency bugs.** `klauspost/compress` and `fasthttp/websocket` behave exactly as documented (an unset memory bound / an unset read limit means "no limit"). Bifrost simply omitted the guard the libraries provide. These go to **Bifrost only**.
- **C4 interacts with `fasthttp/router`'s raw-path matching**, but the router behaves as documented; the defect is that Bifrost authorizes on a *different* (normalized) path than it routes on. Bifrost only.

### 3.2 Report to Bifrost first, or to the dependencies first?

**Report to Bifrost first.** Rationale:

1. **Exposure is concrete and reachable at Bifrost**, unauthenticated, on a deployed gateway. The dependency bugs are only interesting here because Bifrost reaches them with attacker input in no-recover goroutines.
2. **Bifrost can protect users fastest.** Every mitigation — `recover()` wrappers, an SDP size cap, `WithDecoderMaxMemory`, `SetReadLimit`, depth-bounded JSON validation, the four authz fixes — lives in Bifrost's own code and ships on Bifrost's release cadence, with no dependency on an upstream release.
3. **Bifrost is the only party that can address all nine.** Six are pure Bifrost issues; the other three still require Bifrost-side defense-in-depth regardless of the upstream fix.

**Then, in parallel and under embargo, handle the dependency root causes:**

- **Upgrade-first triage.** Before filing anything upstream, check whether a newer `pion/webrtc`, `pion/dtls`, or `gjson` release already fixes A1/A2/A3. If so, the action collapses to a **Bifrost dependency bump** and no upstream report is needed. (The findings are against the *pinned* versions `webrtc v4.2.9`, `dtls v3.1.2`, `gjson v1.18.0`.)
- **If still unfixed upstream, report privately** to pion / tidwall (GitHub private security advisory), ideally with Bifrost co-filing since Bifrost is a direct pion consumer. The pion crashes affect the whole WebRTC-in-Go ecosystem, so a coordinated upstream fix has value beyond Bifrost — but **do not disclose publicly before a fix window**, because public disclosure would expose every pion user, Bifrost included.

**Do not open a public PR / issue with these details until fixes are staged.** Bifrost's defense-in-depth mitigations (recover/caps) can and should land first, because they neutralize the crashes even before the upstream root cause is fixed.

### 3.3 Does the report to Bifrost include the remote DoS findings?

**Yes — all five (A1–A5).**
- A4, A5 are pure Bifrost bugs.
- A1, A2, A3 are dependency-rooted but are only reachable because Bifrost exposes them (unauthenticated realtime; raw request body handed to `gjson.ValidBytes`) inside goroutines with no `recover()`. Bifrost must add defense-in-depth (a process-wide/relay `recover()`, an SDP size cap, depth-bounded validation) **regardless of** the upstream fix. They belong in the Bifrost report, framed as *"reachable via Bifrost → Bifrost mitigation + upstream root cause."*

### 3.4 Which findings go where

| ID | → Bifrost | → Dependency (root-cause report) |
|----|:--------:|----------------------------------|
| A1 | ✅ (recover + SDP size cap) | `pion/webrtc` — `getRids` |
| A2 | ✅ (recover on relay/media goroutines) | `pion/dtls` — `fragmentBuffer.pop()` |
| A3 | ✅ (depth-bounded validate; don't feed raw body to `gjson`) | `tidwall/gjson` — optional recursion cap *(may be treated as by-design; Bifrost fix is the reliable one)* |
| A4 | ✅ (`WithDecoderMaxMemory`) | — (not a klauspost bug) |
| A5 | ✅ (`SetReadLimit`) | — (not a fasthttp/websocket bug) |
| C1 | ✅ | — |
| C2 | ✅ | — |
| C3 | ✅ | — |
| C4 | ✅ | — *(fasthttp/router raw-path behavior is documented; note it as context)* |

### 3.5 Suggested timeline

1. **Day 0:** deliver this report to Bifrost maintainers privately. Land Bifrost defense-in-depth mitigations for A1–A5 and the fixes for C1–C4 (they are independent and low-risk).
2. **Day 0–7:** upgrade-first triage of pion/gjson; if unfixed, open private upstream advisories (co-filed with Bifrost).
3. **Embargo ~90 days** or until fixes ship in both Bifrost and (if applicable) the dependency, whichever first; then coordinated public disclosure / CVE assignment.

---

## 4. Findings — Availability (remote DoS)

Reachability note: on a DEFAULT instance all five are unauthenticated. On a HARDENED instance, A4/A5 remain unauthenticated (they execute before inference auth); A1/A2 require the realtime feature to be enabled; A3 requires the GenAI cached-content route.

### A1 — `pion/webrtc` simulcast SDP slice-bounds panic → full-process crash
- **Severity:** High · **CWE-125 / CWE-248** · unauthenticated
- **Root cause (dependency):** `pion/webrtc/v4@v4.2.9` `sdp.go:301-303` `getRids`:
  ```go
  ridStates := strings.Split(simulcastAttr, ";")   // "send ;" → ["", ""]
  for _, ridState := range ridStates {
      if ridState[:1] == "~" {                      // ""[:1] → panic: slice bounds out of range [:1] with length 0
  ```
  `descriptionIsPlanB` (`peerconnection.go:1203`) calls this **unconditionally**, before any ICE/fingerprint validation.
- **Bifrost path:** `POST /v1/realtime/calls` (or `POST /v1/realtime`, `application/sdp`) → the attacker-controlled SDP offer is passed with no size/format validation to `SetRemoteDescription` at `handlers/webrtc_realtime.go:554`, **synchronously in the fasthttp handler goroutine**.
- **Trigger:** an otherwise-minimal offer whose media section contains `a=simulcast:send ;` (also `a=simulcast:;`) plus `m=audio 9 UDP/TLS/RTP/SAVPF 0`, `a=mid:0`, no `recvonly`/`inactive`. No handshake/ICE needed.
- **Verification:** reproduced against the cached pion source (`panic: runtime error: slice bounds out of range [:1] with length 0`, stack through `getRids → trackDetailsFromSDP → descriptionIsPlanB → SetRemoteDescription`); a control offer with no empty segment does not panic. Independently re-read `sdp.go:303` and `peerconnection.go:1203`.
- **Fix:** *pion* — guard empty `ridState` (`if ridState == "" { continue }`, or `strings.HasPrefix(ridState, "~")`). *Bifrost* — cap SDP body size and wrap `SetRemoteDescription` (and relay/media goroutines) in `recover()`.

### A2 — `pion/dtls` handshake fragment nil-deref → full-process crash
- **Severity:** High · **CWE-476** · unauthenticated (requires completing a standard ICE exchange)
- **Root cause (dependency):** `pion/dtls/v3@v3.1.2` `fragment_buffer.go:111-136` `pop()`. When a handshake header declares `Length=0`, the three completeness guards (`:117`, `:123`, `:132`) all pass trivially and the reassembly loop never runs; line 136 `frags.fragmentByOffset[0].handshakeHeader` then indexes offset `0`, which is absent when the sole buffered fragment sits at a non-zero offset → nil `*fragment` deref. `push()`/`header.Unmarshal` perform no offset-vs-length cross-check.
- **Bifrost path:** `POST /v1/realtime/calls` with `a=setup:active` (forces Bifrost to be the DTLS server); the attacker completes ICE with the credentials Bifrost returns in its SDP answer, then sends the record. Parsing runs in the pion DTLS read goroutine (`conn.go:1199`) — **no `recover()` anywhere in `pion/dtls`**.
- **Trigger:** one 25-byte unencrypted (epoch 0) DTLS record with handshake header `Length=0, FragmentOffset=1, FragmentLength=0`.
- **Verification:** reproduced with a faithful `push`/`pop` harness (nil-deref panic); independently re-read `fragment_buffer.go:111-136`.
- **Fix:** *pion* — in `pop()` bail when `fragmentByOffset[0]` is absent; reject `FragmentOffset != 0 && Length == 0` in `push()`. *Bifrost* — `recover()` on the relay/media goroutines + connection-rate limits.

### A3 — `gjson.ValidBytes` unbounded recursion → stack overflow (uncatchable)
- **Severity:** High · **CWE-674** · unauthenticated
- **Root cause:** `tidwall/gjson@v1.18.0` validator (`gjson.go` `validpayload/validany/validarray/validobject`) is mutually recursive with **no depth limit**; deep nesting drives the goroutine stack past Go's 1 GB limit → `fatal error: stack overflow`, which `recover()` cannot catch. **Bifrost misuse:** `handlers/integrations/genai.go:1170` calls `gjson.ValidBytes(ctx.Request.Body())` on the **raw** request body, so the `sonic` 4096-depth cap (which protects the normal request path) never applies.
- **Bifrost path:** `POST|PATCH /genai/v1beta/cachedContents` (+ the Vertex variant). Route registered unconditionally.
- **Trigger:** a ~6–7 MB body of `[` characters (opening brackets only; under the 100 MB body cap).
- **Verification:** reproduced against cached `gjson v1.18.0` (`runtime: goroutine stack exceeds 1000000000-byte limit → fatal error: stack overflow`); ≤5 M brackets return cleanly, ≥6 M crash. Independently re-read `genai.go:1170`.
- **Fix:** *Bifrost* — validate via a depth-bounded parser (route through `sonic`, which returns a graceful error) or impose a nesting-depth cap before `gjson.ValidBytes`. *gjson* — optionally add a configurable recursion limit (maintainer may treat unbounded recursion as by-design; the Bifrost-side fix is the reliable one).

### A4 — zstd window pre-allocation memory bomb (~10 bytes → 512 MB)
- **Severity:** High · **CWE-789 / CWE-400** · unauthenticated, **pre-auth on every route**
- **Root cause (Bifrost dep misuse):** `core/providers/utils/decompression.go:155,179` call `zstd.NewReader(r, zstd.WithDecoderConcurrency(1))` with **no** `WithDecoderMaxMemory`/`WithDecoderMaxWindow`, so `klauspost/compress@v1.18.6`'s default 512 MiB `maxWindowSize` applies. A zstd frame's Window_Descriptor byte forces `history.ensureBlock()` to allocate the full window **before any output**, so the 100 MB *output* `LimitedReader` never trips.
- **Bifrost path:** `RequestDecompressionMiddleware` wraps the whole router **outside** the per-route auth middleware (`server/server.go:2358`), so decompression runs before routing/auth.
- **Trigger:** `Content-Encoding: zstd` + a ~10-byte body whose window descriptor selects a 512 MiB window. ~5×10⁷ amplification; a few dozen concurrent requests → process OOM.
- **Verification:** re-read `decompression.go:155/179` (no memory bound) and `server.go:2358` (middleware ordering); klauspost default `maxWindowSize = MaxWindowSize` confirmed.
- **Fix:** pass `zstd.WithDecoderMaxMemory(N)` (bounded to e.g. `MaxRequestBodySizeMB`) to both `NewReader` calls.

### A5 — WebSocket unbounded message → OOM full-process crash (missing `SetReadLimit`)
- **Severity:** High · **CWE-400** · unauthenticated
- **Root cause (Bifrost omission):** `fasthttp/websocket@v1.5.12` enforces a size cap only when `readLimit > 0`; `newConn` leaves it `0` (no cap). Bifrost calls `SetReadLimit` only in code-mode (`handlers/websocket.go:99`), **not** on the realtime client connections (`wsresponses.go:139`, `wsrealtime.go:346`), where `ReadMessage → io.ReadAll` grows unbounded. The 100 MB body cap is bypassed after `Hijack`.
- **Bifrost path:** `GET /v1/responses` (or `/v1/realtime`) upgrades with zero auth (`CheckOrigin` returns true with no `Origin`).
- **Trigger:** a WS frame declaring a 2⁴⁴ payload length; stream masked payload until host RAM is exhausted.
- **Verification:** the frame parser itself is clean (negative/overflow lengths rejected, no attacker-sized `make()`); the defect is the missing limit. Re-read the call sites.
- **Fix:** call `ws.SetReadLimit(N)` on the realtime client connections, mirroring code-mode.

---

## 5. Findings — Confidentiality / Integrity (HARDENED instance)

### C1 — Realtime proxy establishes a paid upstream with no virtual key
- **Severity:** High · **CWE-306 / CWE-862** · unauthenticated, HARDENED
- **What:** on a hardened instance (`EnforceAuthOnInference=true`), the WS/WebRTC realtime **proxy** paths open an authenticated upstream to the provider on Bifrost's own paid key **before** the mandatory-VK gate runs, and forward non-turn client events with no governance.
- **Path:** `GET /v1/realtime` / `POST /v1/realtime/calls` with no VK → `InferenceMiddleware` passes through → `RunPreRequestHooks → PreRequestHook` returns `nil` for an empty VK (`plugins/governance/main.go:1210`) → `SelectKeyForProviderRequestType` picks any configured key → `pool.Get` opens the upstream (`handlers/wsrealtime.go:~306`). The `isVkMandatory` 401 gate lives only in `EvaluateGovernanceRequest`, whose sole callers are `PreLLMHook` (main.go:1309), MCP tool execution (1438), and realtime-secret **minting** (`realtime_client_secrets.go:185`) — **never** the realtime proxy connection path.
- **Impact:** an unauthenticated caller forces the gateway to hold provider Realtime sessions on the operator's key and, via a forwarded `session.update` setting `turn_detection: server_vad`, causes **billable** server-generated responses. (Explicit `response.create` turns are still blocked by the per-turn hook.)
- **Proof it's an omission:** the sibling minting path *does* call `EvaluateGovernanceRequest` before minting; the WS Responses API gates events via `RunStreamPreHooks` before the upstream write. Verified callers via grep + re-read of `main.go:1210`, `wsrealtime.go`.
- **Fix:** call `EvaluateGovernanceRequest` before opening the realtime upstream (mirror the minting path).

### C2 — MCP tool-authorization bypass via client-name dash-prefix confusion
- **Severity:** Medium-High · **CWE-863** · one low-priv virtual key, HARDENED
- **What:** two matchers disagree. Discovery/registration uses the **exact** `fmt.Sprintf("%s-*", clientName)` (`core/mcp/utils.go:499`); the governance execution gate `isMCPToolAllowedByVKWith` uses the **loose** `strings.HasPrefix(toolPattern, clientName+"-")` (`plugins/governance/main.go:1172`). `POST /v1/mcp/tool/execute` calls `ExecuteChatMCPTool` directly (`handlers/mcpinference.go`) without the precise include-tools filter, so the loose matcher is the only per-VK gate.
- **Trigger:** given two clients where one name is a dash-prefix of the other (e.g. `note` and `note-taker`, both `tools_to_execute: ["*"]`), a VK granted `note` but **not** `note-taker` sends `{"function":{"name":"note-taker-list"}}` → `HasPrefix("note-taker-list","note-")` is true → allowed → `GetClientForTool` resolves to `note-taker` → executes. Cross-tenant if `note-taker` fronts another tenant's server/credentials.
- **Verification:** re-read the loose matcher (`main.go:1172`), the exact matcher (`utils.go:499`), and the direct-execute path (`mcpinference.go`).
- **Fix:** match the client name at a whole-segment boundary, or run the precise include-tools filter on `/v1/mcp/tool/execute`.

### C3 — MCP user-OAuth token for a VK-less user fails open to the global server
- **Severity:** High · **CWE-863 / CWE-636 (fail-open)** · enterprise MCP-OAuth, HARDENED
- **What:** in `getMCPServerForRequest` (`handlers/mcpserver.go`), the SCIM-header user path **fails closed** when the user has no VK (`:657`, returns an error), but the JWT user-mode path **fails open** — `userScopedServer` returns `(nil, nil)` for the same condition (`:832-834`) and the caller assigns `h.globalMCPServer` (`:767`), which is synced with **all tools from all clients**. Governance `PreMCPHook` then skips the per-VK tool check because `virtualKeyValue == ""` (`main.go:1449`).
- **Trigger:** a logged-in SSO user with no VK completes the OAuth consent selecting `mode:"user"`, then calls `POST /mcp` with the resulting bearer token → reaches the global MCP server (all tenants' tools). Requires `MCPServerAuthMode ∈ {both, oauth}`.
- **Verification:** agent-traced against the cited lines; the SCIM-vs-JWT asymmetry for the identical "no VK" condition is the tell. (Recommend a live repro before upstreaming.)
- **Fix:** mirror the SCIM path — deny when a user-mode token resolves to no VK; never fall back to the global server.

### C4 — Admin-auth bypass via raw-vs-normalized path divergence *(generalization of a known bug)*
- **Severity:** High as an auth-bypass primitive / Medium practical · **CWE-436 / CWE-863** · unauthenticated, HARDENED
- **Relationship to prior work:** this shares its **root cause** with the previously-known `/api/skills/serve` router-backtracking bypass. It is reported here because it **generalizes** that bug — the same divergence defeats admin auth across the *entire* `/api/<segment>/{id}` surface via the exact `/api/version` and `/health` whitelist entries — proving the fix must be systemic, not skills-specific.
- **What:** the router dispatches on the **raw** path (`fasthttp/router@v1.5.4` `router.go:434`, `PathOriginal()` — `%2f` and `..` are literal), while `APIMiddleware` authorizes on the **normalized** path (`handlers/middlewares.go:1015`, `ctx.Path()` — decodes `%2f`→`/`, resolves `..`). `DisablePathNormalizing` is not set.
- **Trigger:** `DELETE /api/webhooks/..%2fversion` → router matches raw `/api/webhooks/{id}` = `deleteWebhookEndpoint` (admin), while auth normalizes to whitelisted `/api/version` and skips. Method-agnostic; reaches `getLog`, `get/deleteProvider`, `deleteVirtualKey`, etc.
- **Impact ceiling (honest):** the captured `{id}` is forced to the escape string `..%2fversion`, which resolves to no real resource, so single-param mutating handlers do lookup-then-404 (none upsert), and list/global/create endpoints are static routes unreachable by the trick. The auth *gate* is genuinely defeated across the surface; the clean high-impact sink with a param-independent side effect remains the skills file handler (the known case).
- **Verification:** **wire-verified** by feeding raw request lines through a real `fasthttp.Server.ServeConn` + `router@1.5.4` + `fasthttp@1.71.0` (`DELETE /api/webhooks/..%2fversion → ADMIN deleteWebhook, ctx.Path()=/api/version`); I independently confirmed the whitelist entries, the two path sources, and the `webhooks/{id}` route.
- **Fix:** authorize on the same normalized path the router dispatches on (or set `DisablePathNormalizing=true` and reject `%2f`/`..`/`\`). A defense-in-depth check that rejects requests where `PathOriginal()` ≠ normalized path closes the whole class.

---

## 6. Negatives & assurance (investigated, not found)

- **Non-admin / dependency-driven RCE — none found.** The skills `git upload-pack` subprocess uses a compile-time-constant argv, an empty env (the `Git-Protocol` header is not propagated), and an in-memory object store (no on-disk `.git/config`/hooks), so there is no argument/option/hook injection. Code-mode Starlark exposes only admin-configured tool builtins (no `os`/`file`/`unsafe`, `load()` disabled, no introspection escape). CEL declares variables only. MCP stdio/`.so`/postgres `password_command` are all admin-sourced. No `yaml`/`gob`/`xml`/`text-template` runs on attacker data. *(Residual, not a claim: `sonic`'s hand-written assembly parses every request body — a memory-safety bug there would be RCE, but no concrete primitive was found.)*
- **LFI / unauthorized local-file read — none found on a hardened instance.** Inference media fetch is http(s)-only behind `SSRFSafeDialContext`; UI serving is `embed.FS`; there is no archive *extraction* anywhere (only zip writing); all `file://` loaders are admin-config-gated; the object store has no local-filesystem backend; MCP exposes no `resources/read`. A virtual key never authorizes `/api/*`.
- **MCP credstore cross-tenant (VK holder) — securely bound;** MCP resource LFI/SSRF — closed; MCP connection SSRF — admin-config only.
- **Session/admin token forgery, cookie→admin, temp-token→admin, CL/TE smuggling (self-contained) — ruled out.**

### Hardening items (not exploitable today)
- Skills `storage_key` handler validates the `skills/` prefix only for `upload` type, not `text`/`dataurl` — compensated by a stricter store-layer scope check and admin-only writes, but latent defense-in-depth debt. `validateSkillStorageKeyScope` uses `strings.HasPrefix` without `path.Clean` (inert on real S3/GCS; risky on a path-normalizing gateway).

---

## 7. Appendix — draft disclosures

> Fill in `<contact>`/dates before sending. Send A first; send B–D only after upgrade-first triage confirms the pinned version is still affected.

### A. Private report → Bifrost maintainers (security contact)

> **Subject:** [Security] 9 findings in Bifrost (5 remote DoS incl. dependency crashes, 4 authz/VK bypass on hardened deployments)
>
> Hi — a dependency-focused security review of Bifrost `dev`@`2c668cd3b` (V2.0.0) surfaced nine confirmed, undisclosed issues. Full technical detail (triggers, code paths, verification, fixes) is attached as `SECURITY_FINDINGS.md`. Summary:
>
> - **5 remote DoS**, all unauthenticated, each crashing the whole process (no fasthttp `PanicHandler`): a `pion/webrtc` SDP panic (A1) and a `pion/dtls` nil-deref (A2) reachable via the realtime endpoints; a `gjson` stack overflow via `/genai/v1beta/cachedContents` (A3); a zstd 512 MB pre-allocation via `Content-Encoding: zstd` (A4); and an unbounded WebSocket message on `/v1/responses` (A5). A1–A3 are dependency-rooted but reachable through Bifrost and need Bifrost-side defense-in-depth (`recover()`, size/depth caps); A4/A5 are missing decoder/read limits in Bifrost.
> - **4 authorization issues on hardened instances** (`EnforceAuthOnInference=true`, admin creds set): a no-VK caller drives billable Realtime upstreams (C1); an MCP tool-authz dash-prefix bypass (C2); an MCP user-OAuth fail-open to the global server (C3); and a systemic admin-auth bypass via raw-vs-normalized path handling that generalizes the known `/api/skills/serve` issue (C4).
>
> Recommended immediate mitigations (all in Bifrost, no upstream dependency): wrap `SetRemoteDescription` and the realtime relay/media goroutines in `recover()` + cap SDP size; `zstd.WithDecoderMaxMemory`; `ws.SetReadLimit` on realtime conns; depth-bounded validation before `gjson.ValidBytes`; the four authz fixes in §4–5. We suggest a ~90-day embargo and can help coordinate the upstream pion/gjson reports.

### B. Private advisory → `pion/webrtc` (GitHub Security Advisory)

> **Title:** Slice-bounds panic in `getRids` when parsing a malformed `a=simulcast` SDP attribute
>
> `SessionDescription.Unmarshal` / `SetRemoteDescription` reaches `getRids` (`sdp.go`), which does `strings.Split(simulcastAttr, ";")` and then `ridState[:1]` without guarding empty segments. An offer containing `a=simulcast:send ;` (or `a=simulcast:;`) yields an empty segment and panics with `slice bounds out of range [:1] with length 0`. `descriptionIsPlanB` calls this before ICE/fingerprint validation, so a remote peer can crash any consumer that calls `SetRemoteDescription` on an untrusted offer. Affected: v4.2.9 (please confirm latest). Fix: skip empty `ridState` or use `strings.HasPrefix(ridState, "~")`. PoC offer and stack trace attached.

### C. Private advisory → `pion/dtls` (GitHub Security Advisory)

> **Title:** Nil-pointer dereference in `fragmentBuffer.pop()` on a zero-length handshake fragment at a non-zero offset
>
> A DTLS handshake header with `Length=0` and `FragmentOffset≠0` passes all completeness checks in `pop()` (`fragment_buffer.go`), which then dereferences `fragmentByOffset[0]` — absent — causing a nil-pointer panic in the record read goroutine (no `recover()` in the package). A single unencrypted (epoch 0) 25-byte record triggers it pre-authentication during the handshake. Affected: v3.1.2 (please confirm latest). Fix: bail in `pop()` when the offset-0 fragment is missing, and reject `FragmentOffset≠0 && Length==0` in `push()`/header parsing. PoC bytes attached.

### D. Issue / advisory → `tidwall/gjson`

> **Title:** `Valid`/`ValidBytes` can stack-overflow (fatal, unrecoverable) on deeply nested input
>
> The validator (`validpayload`/`validany`/`validarray`/`validobject`) recurses per nesting level with no depth limit; ~6M nested `[` triggers `fatal error: stack overflow`, which `recover()` cannot catch. Consumers that validate untrusted input are exposed to a remote crash. Consider an optional configurable recursion/depth limit. (Reported here for awareness; downstream users should also bound nesting before calling `Valid*`.)

---

*Prepared as part of an authorized security assessment of the maintainer's own repository. Keep confidential until coordinated disclosure completes.*
