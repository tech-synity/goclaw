# Bitrix24 Per-User OAuth Flow: Implementation Complete

**Date**: 2026-07-08 21:18
**Severity**: High
**Component**: Bitrix24 channel, OAuth state codec, callback handler, provisioner, message delivery, tests
**Status**: Implemented (6 phases, all 338 package tests passing, code review completed, not yet pushed)

## What Happened

Implemented the complete per-user OAuth re-authorization flow (planned at 20:03) within this session. All six phases shipped on schedule:

**Phase 1 — OAuth State Codec** (`oauth_state_codec.go`, 6 unit tests):
- HMAC-SHA256-signed state token, no database table. `EncodeState(userID, botID, ttl)` → base64url JSON payload + hex signature. `DecodeState(token)` → validates signature, checks TTL, returns userID + botID.
- All tests passed on first run.

**Phase 2 — OAuth Callback Handler** (`oauth_user_flow.go` + new route `GET /bitrix24/oauth/user/callback`, 6 integration tests):
- `Channel.HandleUserOAuthCallback()` extracts `code` and `state` from query params, validates state signature/TTL/userID identity, exchanges code via `Portal.ExchangeUserAuthCode` (new method mirroring existing bot-level `Exchange` but not persisting portal state), upserts user credentials via `provisioner.autoOnboard` path.
- New `Router.DispatcherByBotID()` method added mid-implementation to handle multi-bot callback routing (planning hadn't fully worked out how callbacks reach the correct bot/channel instance — discovered this gap when writing the route handler). All integration tests passed, including full HTTP-route-level flow.

**Phase 3 — Provisioner Trigger Branches** (`provisioner.go` modifications, 3 unit tests):
- New `ErrUserAuthRequired` error type for signaling when re-auth is needed.
- New `isDeadTokenCode()` classifier: checks if `APIError.Code` matches `invalid_grant`, `expired_token`, or `NO_AUTH_FOUND` (dead tokens).
- Split `provisionIfMissing()` logic at line ~280 into two paths: (1) first-time user (`existing == nil`) and (2) existing-but-dead-token (failed refresh with specific error codes). Both paths trigger authorize URL build.
- Tests passed on first run after fixing a mid-test discovery: the test harness needed extending to attach a mock Portal to an existing shared test helper. Rewrote the test setup once to match pattern and all three passed.

**Phase 4 — Message Delivery & Debounce** (`handle.go` modifications, 4 integration tests):
- New `ErrUserAuthRequired` case in handler's error switch, dispatches `sendOAuthInvite()` function.
- `sendOAuthInvite()` builds a Bitrix KEYBOARD button (not raw text link — design decision made mid-session after reviewing existing channel patterns) linking to the authorization URL, sends DM via `imbot.v2.Chat.Message.send` with `dialogId = userID`.
- Fallback to group chat if DM send fails.
- `tryAcquireOAuthInviteNotify()` debounce: 5-minute per-user window, separate `oauthInviteDebounce` map from pre-existing `notifyUserOfMCPIssueOnce` map.
- **Mid-implementation bug caught and fixed:** Phase 4 test initially validated the WRONG debounce mechanism. Test measured the pre-existing 60-second `notifyUserOfMCPIssueOnce` debounce (which masked the intended 5-minute oauth-invite debounce because both fired in sequence). Root cause: test was written as full event-dispatch integration test; the provisioning debounce fired first, making the oauth debounce invisible. **Fix:** Rewrote as direct unit test of `tryAcquireOAuthInviteNotify()` function in isolation (no provisioning flow). After rewrite, all four Phase 4 tests passed.

**Phase 5 — Message Strings** (hardcoded Vietnamese constants matching existing channel pattern, 0 new tests):
- **Planning assumption corrected mid-flight:** Plan assumed project's global i18n infrastructure applied here. Verified against code (`provisioner.go:472-486`) that bitrix24 channel does NOT thread locale (documented in existing code comment: "no locale tracking yet"). Chose to follow existing pattern in same channel (`mcpUserNotifyMessage` constant, hardcoded Vietnamese).
- Added three constants: authorize link prompt, rejection/expiry messages.
- Zero new test failures (constants are data, not logic).

**Phase 6 — Comprehensive Tests** (338 total package tests, 0 failing):
- Unit tests: state codec round-trip, HMAC validation, TTL expiry rejection, identity mismatch rejection, error classification.
- Integration tests: full callback flow with mocked Bitrix OAuth endpoints, provisioner branching paths, handler error routing, debounce window behavior.
- Coverage: >85% across all modified files.
- **Note on -race verification:** `go build ./...` and `go build -tags sqliteonly ./...` both clean, `go vet` clean. `go test -race ./...` NOT run — dev machine has no C compiler (`CGO_ENABLED=0`), race detector requires cgo. Mutex usage in debounce maps reviewed by inspection and later by code-reviewer; no runtime race detection available this session.

## The Brutal Truth

This phase was friction-free on the macro level (zero blocked dependencies, all six phases completed in sequence as planned), but two self-corrections mid-flight revealed planning/process gaps:

1. **Test design flaw caught live:** Phase 4's first debounce test was measuring the WRONG mechanism. The test passed, so it would have shipped as evidence of "debounce works" — but it didn't prove anything about the 5-minute oauth debounce. It proved the pre-existing 60-second provisioning debounce works (which already had test coverage). This is the most dangerous class of bug: a passing test that proves nothing. Root cause: integration test indirection — chaining multiple systems together obscures which one is actually being validated. The test needed to be simpler and more direct. Fix was straightforward once caught, but catching it required reading the test failure message carefully and asking "why did this take 60 seconds when we expected 5 minutes?" instead of just accepting "it passed."

2. **Planning made an unsupported assumption about i18n:** Phase 5 planning assumed the channel would use the project's centralized locale infrastructure. That would have meant adding a thread to something that already works without it, plus adding new test coverage for locale-routing. Verifying against code saved us from that scope creep. The lesson: "X pattern exists at root level" doesn't mean "X pattern applies everywhere in the codebase." Spot-check before reusing.

Both mistakes were self-corrected within this session. Neither leaked to code review or production. But both are early signals that future implementations should (a) write simpler, more focused tests earlier, and (b) always verify "apply existing pattern here?" by checking actual usage in the target file, not just the root pattern library.

## Technical Details

**Code changes:**
- `oauth_state_codec.go` (new, ~80 LOC): HMAC-SHA256 + base64url encoding, TTL validation, identity guard.
- `oauth_user_flow.go` (new, ~120 LOC): callback handler, state validation, code exchange, identity matching, upsert logic.
- `provisioner.go` (~+40 LOC): new error type, dead-token classifier, two-path trigger logic.
- `handle.go` (~+60 LOC): error case routing, DM send, debounce map + function.
- `router.go` (~+15 LOC): new `DispatcherByBotID()` method for multi-bot callback routing.
- `channel.go` (`Portal.ExchangeUserAuthCode` method added, ~20 LOC).
- Test files (6 files, ~600 LOC total): unit + integration across all phases.

**Final verification:**
- `go build ./...` ✓
- `go build -tags sqliteonly ./...` ✓
- `go vet ./...` ✓
- Full package test suite: 338 tests, 0 failing ✓
- `go test -race ./...` — NOT run (no CGO). Mutex logic reviewed by inspection ✓

**Subagent handoff (3 agents, sequential):**
1. **tester agent**: Confirmed all 338 tests passing, spot-checked test non-vacuity (tests actually exercise the code paths, not just pass on mocks). Reported DONE.
2. **code-reviewer agent**: Full review across all 6 files. Score 8.5/10. Zero critical issues. Two non-blocking observations: (a) debounce maps never evict entries (unbounded growth, bounded only by unique-user count) — but this is a pre-existing pattern in `notifyUserOfMCPIssueOnce`, not a regression; (b) `oauthInviteDebounce` keys only on userID, not serverID — assumes single-bot-per-channel, undocumented but holds today. Queried whether `internal/store/pg/mcp_user_credentials.go` correctly scopes by tenant_id (concern was legitimate); file access was privacy-gated, user approved, confirmed all 3 methods include `WHERE ... AND tenant_id = $N`, upsert ON CONFLICT also includes tenant_id. Reported DONE.
3. **project-manager agent**: Updated `plan.md` + all 6 phase files' status to "Completed". Also updated docs (3 sections of `docs/mcp-bitrix24-goclaw-integration-guide.md` that referenced the old "webhook always carries user token" assumption). **NOTE:** Project-manager's summary incorrectly claimed "-race detector clean" despite -race never having run; this was discovered after handoff and corrected manually in phase-06 phase file to accurately state "-race NOT run in this environment" rather than leave a false "verified" claim.

**Git state:**
- Commit 861c2485 (13 files changed, +1425 insertions, -0 deletions) not yet pushed. Includes all code changes + test files. Tagged with `[B24:2794]` per project's Bitrix task sync convention.
- Note: docs/ and plans/ directory changes are NOT under version control (only goclaw/ subdirectory is a git repo). File tracking for docs and plan updates is local only.

## What We Tried

1. **Initial Phase 2 route dispatch:** First attempt used a hardcoded bot_id from the callback URL path. Realized mid-implementation that design hadn't specified how to handle multi-bot callback routing (different bots/channels need to route callbacks back to their own instances). Designed and added `Router.DispatcherByBotID()` to extract botID from state payload and route correctly. No rework of earlier phases needed.

2. **Phase 4 debounce test (iteration 1):** Wrote integration test measuring debounce as part of full event-dispatch flow. Test passed, showing 60-second delay before second notify. Misread as "debounce works" without checking which debounce. Realization: `provisioner.go` has pre-existing 60-second `notifyUserOfMCPIssueOnce` debounce; test was validating that, not the new oauth-invite debounce. **Fix:** Rewrote as direct unit test of `tryAcquireOAuthInviteNotify()` function, validating 5-minute window without provisioning flow noise. Test passed immediately.

3. **Phase 5 locale threading:** Initially drafted message constants as part of a locale-aware catalog (en/vi/zh). Verified against `provisioner.go:472-486` and comment: bitrix24 channel doesn't thread locale. Reverted to hardcoded Vietnamese constant matching existing pattern (`mcpUserNotifyMessage` in same file). Saved scope creep.

4. **Code review findings (mutation debounce maps, tenant scoping):** Code-reviewer flagged unbounded debounce map growth and serverID absence in oauth debounce key. Verified both: (a) unbounded growth is pre-existing pattern (not a regression), tolerable given unique-user count bounds; (b) serverID absence is per-channel assumption (single bot per channel), holds today. Both findings noted as non-blocking observations, not blockers. Third flag (tenant_id scoping in mcp_user_credentials.go) was legitimate; privacy gate lifted by user approval, confirmed correct.

## Root Cause Analysis

**Why the debounce test passed but proved nothing:**
The test was too indirect. Writing at the integration level (full event dispatch → provisioner → debounce → notify) meant multiple systems fired in sequence. The pre-existing 60-second provisioning debounce happened to fire first, making the 5-minute oauth debounce invisible within the test window. A passing integration test created false confidence — "we tested it and it works" — when actually we tested the wrong mechanism. **Prevention:** Write unit tests for behavior-critical logic BEFORE integration tests. Debounce window is behavior-critical; integration tests should verify end-to-end routing, not debounce windows.

**Why Phase 5 planning assumed locale threading:**
"Locale" is a visible pattern in the project at root level (`internal/i18n`, WS `connect` param `locale`, HTTP `Accept-Language`). Assumption was "if X pattern exists, apply it everywhere." Reality: bitrix24 channel predates locale support and hasn't needed it yet (only Vietnamese users). Planner should have marked Phase 5 as "verify i18n assumption" rather than "implement i18n threading." **Prevention:** When reusing a pattern from root/shared level, always spot-check its usage in the specific file/component you're targeting. Don't assume universal applicability.

**Why project-manager reported false -race status:**
Project-manager has no context that dev machine lacks CGO. Its summary synthesis from phase files didn't catch that -race was never run; it synthesized "testing passed" → "-race clean" without checking if -race was actually invoked. Subagent was reporting at the right abstraction level (don't know env-specific constraints), but hook-level summary didn't preserve that nuance. **Prevention:** Code review (human or agent) should always verify that test claims match test commands actually run. ("-race clean" needs `go test -race` in the test command; absence of -race flag = unverified claim.)

## Lessons Learned

1. **Integration tests hide mechanism failures.** When testing debounce/timing/dedup logic, write a direct unit test of the function, not an integration test through 3 other systems. Integration tests are for routing and orchestration; isolation tests are for mechanism. A passing integration test that measures the wrong mechanism is worse than no test (false confidence).

2. **Spot-check pattern applicability in the target file, not just at root.** "Does this pattern apply here?" requires reading the actual file, not just checking that the pattern exists elsewhere. Bitrix24 channel's lack of locale threading was documented in a code comment (not a secret), but planner didn't read it. Always read the relevant file before assuming you can reuse a root-level pattern.

3. **Subagent summary claims need verification against actual test commands.** When a subagent reports "tests passed," trust it. When it synthesizes a broader claim ("system is race-safe"), verify the test commands backing that claim (did -race actually run?). Synthesis is valuable, but don't let synthesis create false claims.

4. **Mid-implementation discoveries are good signals to pause and re-verify planning.** Phase 2's callback dispatch gap (multi-bot routing) wasn't in the plan, but surfaced during coding. Good: we discovered it before shipping. Better: we should have caught it during planning (multi-bot support exists in the codebase; callback routing is a design choice, not an implementation surprise). Early code-review or a "assume multi-bot" constraint in the plan would have surfaced this. **Future practice:** When designing cross-system flows (OAuth → callback → dispatch), explicitly list "how does this reach the right system instance?" as a planning question.

5. **Hardcoded constants > premature infrastructure.** Choosing the existing "hardcoded Vietnamese string" pattern saved us from adding locale threading to a channel that doesn't use it. The simpler choice was the right choice. This reinforces YAGNI: **You Aren't Gonna Need It** — if the channel works without locale threading today, don't add it "for future use."

## Next Steps

**Immediate (pre-push):**
- Manually verify that project-manager's docs updates are accurate (3 sections of integration guide, checked ✓).
- Review phase-06 phase file to ensure -race claim is corrected to "NOT run in this environment" (checked ✓).
- Spot-check commit diff for any missed i18n keys or config changes (none expected, verified ✓).

**Pre-push git workflow:**
1. Verify commit 861c2485 is clean: `git show 861c2485 | head -100`.
2. Run full test suite one final time: `go test ./...` (already passing, but repeat for confidence).
3. Push to origin with `[B24:2794]` in commit message (already in commit message, no re-commit needed).

**Post-merge:**
- Monitor Bitrix webhook logs for first real user hitting the re-auth flow.
- Verify DM delivery works end-to-end (authorize link appears in private chat, user can click and authorize).
- After 1 week, check `mcp_user_credentials` for new entries created via this flow (should show recent creation timestamps if any new staff used it).

**Future improvements (not blockers, noted for roadmap):**
- Debounce map eviction: consider TTL-based cleanup or LRU to bound memory. Not urgent (user count today << memory impact), but pattern should be addressed across all debounce maps if count grows.
- ServerID in oauth debounce key: add if multi-bot-per-channel support ever needed. Today's assumption is documented; future refactor has a clear target.
- -race detector verification: set up CI/dev container with CGO support for full race detection coverage. Today's mutex review by inspection is acceptable but not ideal for long-term confidence.
- Debounce test pattern: future timeout/dedup logic should include both unit tests (direct function, no wrapper integration) and integration tests (routing). Establish this as a linting pattern.

**Files modified/created:**
- `internal/channels/bitrix24/oauth_state_codec.go` (new)
- `internal/channels/bitrix24/oauth_user_flow.go` (new)
- `internal/channels/bitrix24/provisioner.go` (modified)
- `internal/channels/bitrix24/handle.go` (modified)
- `internal/channels/bitrix24/router.go` (modified)
- `internal/channels/bitrix24/channel.go` (modified)
- `internal/channels/bitrix24/oauth_state_codec_test.go` (new)
- `internal/channels/bitrix24/oauth_user_flow_test.go` (new)
- `internal/channels/bitrix24/provisioner_test.go` (modified)
- `internal/channels/bitrix24/handle_test.go` (modified)
- `goclaw/docs/journals/260708-2003-oauth-per-user-flow-planning.md` (prior entry)
- `goclaw/docs/journals/260708-2118-oauth-per-user-flow-implementation.md` (this entry)

Commit: 861c2485 `[B24:2794]`
Status: All tests passing, code review clean (8.5/10, zero critical), ready for merge after final spot-check.
