# AW-2865 Phase 7 — Research

Phase 7 is the deliverable creation phase: its entire output is a set of per-task Markdown files under
`specs/.current/AW-2865/tasks/` plus a `README.md` index. This research document serves two purposes:
(1) confirms that every line-number citation in `phase-7/tasks.md §2` is still accurate against the
current source files, and (2) provides a recommendation on the task 32 severity escalation question.
No code is changed.

---

## Phase Scope

**Active phase:** 7 — Compile Findings into Per-Task Markdown Files.

This phase consumes the deduplicated Consolidated Findings Inventory (tasks.md §2) and produces 34
standalone Markdown task files under `specs/.current/AW-2865/tasks/`. The only code-adjacent work is
reading source files to verify citation accuracy; no edits are made to `lib/`, `test/`, or `example/`.

Prior phases (1–6) are prerequisites and are treated as frozen inputs here.

---

## Resolved Questions

1. **Row count after merge.** After merging row 22 (duplicate `add-structured-observability-hooks`) into
   row 13, the HIGH section drops from 21 to 20 entries. IMPROVEMENT rows 23–35 remain 13 entries.
   Total deliverable: 1 BLOCKER + 20 HIGH + 13 IMPROVEMENT = **34 task files**. Row 22 produces no
   file; the gap at number 22 is intentional and no re-numbering is done.

2. **Row 22 merge destination.** Row 22 merges INTO row 13. The surviving file is
   `13-add-structured-observability-hooks.md`; it cites cross-reference IDs P6-5, P6-6, P6-8 (original
   row 13) and T-X1 note (row 22).

3. **File numbering.** Original numbers from the Consolidated Findings Inventory are kept. There is a gap
   at 22; files go …21, 23, 24… No renumbering.

4. **Task 32 severity escalation.** See the dedicated section below with evidence and recommendation.

5. **Line number verification.** See the Citation Verification section below.

6. **Tasks directory location.** `specs/.current/AW-2865/tasks/` — a sibling of the `phase-N/` folders,
   not inside `phase-7/`.

---

## Related Modules / Services

The four source files under `lib/src/` are the only implementation artefacts. Layer order (bottom-up):

| File | Lines | Role |
|------|-------|------|
| `lib/src/bhttp.dart` | 191 | RFC 9292 Binary HTTP — varint codec + serialize/parse |
| `lib/src/hpke.dart` | 314 | RFC 9180 HPKE Base Mode Sender only |
| `lib/src/ohttp.dart` | 258 | RFC 9458 KeyConfig parse/validate + encap/decap |
| `lib/src/ohttp_client.dart` | 177 | High-level client — orchestrates the full OHTTP flow |

Public re-export: `lib/ohttp_dart.dart` (single entry point).

Test files mirror the source one-to-one:

| File | Lines | Coverage shape |
|------|-------|---------------|
| `test/hpke_test.dart` | 297 | RFC 9180 App A.1 vectors + RFC 5869 HKDF |
| `test/bhttp_test.dart` | 232 | Varint round-trips + serialize/parse happy path |
| `test/ohttp_test.dart` | 165 | KeyConfig parse/validate + encap structure + decap short-circuit |

No integration tests exist. No `test/integration/` directory exists.

---

## Current Endpoints & Contracts

### OhttpGatewayConfig (ohttp_client.dart lines 35–53)

```
OhttpGatewayConfig({
  required String gatewayBaseUrl,    // no scheme validation
  required String configPath,
  required String requestPath,
  required String targetAuthority,
  String targetScheme = 'https',
  String? directBaseUrl,
})
String get effectiveDirectBaseUrl => directBaseUrl ?? gatewayBaseUrl;
```

`effectiveDirectBaseUrl` is a **public** getter. When `directBaseUrl` is null (the common case — it is
optional with no required constraint), it silently returns `gatewayBaseUrl`. This is the mechanism
driving both task 01 (BLOCKER) and task 32 (IMPROVEMENT / escalation candidate).

### OhttpClient.send() flow (ohttp_client.dart lines 69–145)

1. GET KeyConfig — `_httpClient.get(Uri.parse(...))` — no timeout, no cache (lines 81–83).
2. `OhttpKeyConfig.parse(configResponse.bodyBytes)` — no size cap before parse (line 89).
3. `bhttp.serializeRequest(...)` with `gateway.targetAuthority` (lines 97–104).
4. `ohttpEncapsulate(config, binaryRequest)` (line 108).
5. POST to gateway — `_httpClient.post(...)` — no timeout (lines 115–119).
6. `ohttpDecapsulate(...)` — no size cap on `gatewayResponse.bodyBytes` before passing (lines 129–133).
7. `bhttp.parseResponse(binaryResponse)` (line 136).
8. Header map built with raw `e.key` (no explicit lowercase) (line 139).

### OhttpClient.sendDirect() (ohttp_client.dart lines 147–171)

Single doc-comment line: `"Send a direct HTTP request (for comparison)."` No `@Deprecated`, no assert,
no privacy warning. Uses `effectiveDirectBaseUrl` (line 154).

### OhttpKeyConfig.parse / validate (ohttp.dart lines 32–97)

`parse()` reads exactly the first KDF+AEAD pair from the symmetric algorithms section and returns,
ignoring any remaining pairs (lines 67–78). `parse()` throws `FormatException` for unsupported KEM
(line 51). `validate()` throws `UnsupportedError` for unsupported KEM (lines 82–85) — two different
exception types for the same conceptual error.

### ohttpEncapsulate (ohttp.dart lines 131–167)

`ctx.seal(Uint8List(0), binaryRequest)` — empty AAD is intentional (line 149, comment on line 148).
`exportedSecret` is stored in `OhttpEncapsulateResult` (lines 105–120) and returned to the caller.

### ohttpDecapsulate (ohttp.dart lines 174–226)

Uses plain (unlabeled) HKDF via `HpkeSender.hkdfExtract` / `hkdfExpand` (lines 193, 196, 203). No
maintenance-hazard comment warning against replacing with labeled variants. AES-GCM decrypt at lines
217–225 can throw `SecretBoxAuthenticationError` (a `package:cryptography` internal type) directly to
the caller.

### HpkeSenderContext fields (hpke.dart lines 258–263)

```dart
final Uint8List enc;
final Uint8List key;          // 16 B AES-128-GCM key
final Uint8List baseNonce;    // 12 B nonce
final Uint8List exporterSecret; // 32 B
int _seq = 0;
```

All four `Uint8List` fields are `final` and retained for the object lifetime. Dart GC does not
guarantee prompt zeroing. No zeroization API exists.

---

## Citation Verification

All line numbers were verified against the current source files as they exist on branch
`feature/AW-2865-investigation-ohttp_dart`. File lengths are noted for context.

### ohttp_client.dart (177 lines)

| tasks.md citation | Verified content | Status |
|-------------------|-----------------|--------|
| `:43–51` (task 02 HTTPS enforcement) | Lines 43–51 are the `OhttpGatewayConfig` constructor body (opening `const OhttpGatewayConfig({` through the last field `this.directBaseUrl,`). The closing `})` is line 50, `;` ends line 50. Line 52 is `effectiveDirectBaseUrl`. The citation is valid for the constructor; the companion getter is at 52. | CONFIRMED — include `:52` as well |
| `:52` (tasks 01, 32) | `String get effectiveDirectBaseUrl => directBaseUrl ?? gatewayBaseUrl;` | CONFIRMED |
| `:69–89` (task 09 KeyConfig caching) | Lines 69–89: `send()` signature through `OhttpKeyConfig.parse(...)`. The GET call is at 81–83, the status check at 84–88, the parse at 89. Covers the full KeyConfig-fetch block. | CONFIRMED |
| `:81–83` (tasks 08, 13) | `final configResponse = await _httpClient.get(Uri.parse(...));` — exactly 3 lines (81, 82, 83). | CONFIRMED |
| `:82` (task 25 URL normalization) | `Uri.parse('${gateway.gatewayBaseUrl}${gateway.configPath}')` — line 82 is the argument inside `.get(...)`. | CONFIRMED |
| `:84–87` (task 10 typed errors) | Lines 84–87: `if (configResponse.statusCode != 200) { throw Exception(...); }` | CONFIRMED |
| `:97–104` (task 03 targetAuthority restriction) | Lines 97–104: `bhttp.serializeRequest(method:..., authority: gateway.targetAuthority, ...)` call. | CONFIRMED |
| `:115–119` (tasks 08, 13) | Lines 115–119: `final gatewayResponse = await _httpClient.post(Uri.parse(...), headers:..., body:...);` | CONFIRMED |
| `:116` (task 25 URL normalization) | `Uri.parse('${gateway.gatewayBaseUrl}${gateway.requestPath}')` — line 116. | CONFIRMED |
| `:120–122` (task 10 typed errors) | Lines 120–122: `if (gatewayResponse.statusCode != 200) { throw Exception(...); }` | CONFIRMED |
| `:129–133` (task 12 size cap) | Lines 129–133: `final binaryResponse = await ohttpDecapsulate(encResult.enc, encResult.exportedSecret, gatewayResponse.bodyBytes);` | CONFIRMED |
| `:139` (task 26 lowercase headers) | Line 139: `.map((e) => OhttpHeader(name: e.key, value: e.value))` — `e.key` is the raw key from `streamedResponse.headers.entries`; no forced lowercase. | CONFIRMED |
| `:148–171` (tasks 01, 13) | Lines 147–171 is `sendDirect()`. The doc comment is line 147; the method body runs 148–171. The citation at `148–171` correctly targets the method body. | CONFIRMED (off by one on opening line: method body starts at 148 not 147) |
| `:154` (task 25 URL normalization) | `final uri = Uri.parse('${gateway.effectiveDirectBaseUrl}$path');` — line 154. | CONFIRMED |

### ohttp.dart (258 lines)

| tasks.md citation | Verified content | Status |
|-------------------|-----------------|--------|
| `:51` (task 23 unify KEM exception) | `throw FormatException('Unsupported KEM: ...')` in `parse()` | CONFIRMED |
| `:60–78` (task 06 silent KDF+AEAD drop) | Line 60: `final symLen = ...`. Lines 63–65: bounds check. Lines 67–71: read first KDF+AEAD pair. Lines 72–78: `return OhttpKeyConfig(...)`. The parser exits after reading exactly one pair, ignoring any remainder. | CONFIRMED |
| `:82–85` (task 23 unify KEM exception) | `validate()` UnsupportedError for KEM at lines 82–85. | CONFIRMED |
| `:105–120` (task 05 zeroize exportedSecret) | `OhttpEncapsulateResult` class with `enc`, `encRequest`, `exportedSecret` fields and constructor (lines 105–120). The `exportedSecret` field is defined at line 113. | CONFIRMED |
| `:148–150` (task 28 expand empty-AAD comment) | Line 148: `// Seal with empty AAD (per reference Go implementation, not RFC header)`. Line 149: `final ct = await ctx.seal(Uint8List(0), binaryRequest);`. Line 150 is blank. The comment exists but does not cite RFC 9458 §4.3. | CONFIRMED |
| `:192–207` (task 29 plain-HKDF comment) | Lines 192–207 cover the plain-HKDF block in `ohttpDecapsulate`: line 192 is the `// prk = HKDF-Extract(salt, secret) — plain HKDF, not labeled` comment, line 193 is `final prk = await HpkeSender.hkdfExtract(...)`, lines 195–200 are the `aeadKey` HKDF-Expand, lines 202–207 are the `aeadNonce` HKDF-Expand. No warning comment about labeled vs. plain substitution hazard. | CONFIRMED |
| `:196–207` (task 05 zeroize exported secret) | Lines 196–207 are the HKDF key/nonce derivation block inside `ohttpDecapsulate`, which consumes `exportedSecret`. | CONFIRMED |
| `:217–225` (task 07 wrap AEAD error) | Lines 217–225: `AesGcm.with128bits()` instantiation through `aesGcm.decrypt(...)`. `decrypt` can throw `SecretBoxAuthenticationError` from `package:cryptography` — this type is not re-exported by `lib/ohttp_dart.dart`. | CONFIRMED |

### hpke.dart (314 lines)

| tasks.md citation | Verified content | Status |
|-------------------|-----------------|--------|
| `HpkeSenderContext field declarations` (task 04) | `HpkeSenderContext` class begins at line 258. Fields `enc` (259), `key` (260), `baseNonce` (261), `exporterSecret` (262), `_seq` (263). All four `Uint8List` fields are `final` with no zeroization. | CONFIRMED |

### bhttp.dart (191 lines)

| tasks.md citation | Verified content | Status |
|-------------------|-----------------|--------|
| `:44–67` (task 11 bounds-check) | Lines 44–67: `decodeVarint` function. `data[offset]` access at line 45 (no bounds check); `data[offset + 1]` at line 51; `data[offset + 1..4]` at lines 54–55; loop `data[offset + i]` at line 61. All throw `RangeError` on truncated input. | CONFIRMED |
| `:74–111` (task 24 serialize size cap) | Lines 74–111: `serializeRequest` function. No size cap on `method`, `scheme`, `authority`, `path`, `headers`, or `body` before varint encoding. | CONFIRMED |
| `:148–162` (task 27 status code range) | Lines 148–162: response parsing loop. `statusCode` is read as a raw varint (line 152/153) and checked only for `100 <= x < 200` (informational skip). No validation that the final `statusCode` is in a valid HTTP range (100–599). | CONFIRMED |
| `:164–187` (tasks 11, 12 size cap + bounds) | Lines 164–187: header-section + content parsing. `headersLen` at line 165 has no upper bound. `data.sublist(offset, offset + contentLen)` at line 187 has no upper bound on `contentLen`. | CONFIRMED |
| `:173` (task 11 RangeError) | Line 173: `data.sublist(offset, offset + nameLen)` — throws `RangeError` on truncated input. | CONFIRMED |
| `:178` (task 11 RangeError) | Line 178: `data.sublist(offset, offset + valueLen)` — throws `RangeError` on truncated input. | CONFIRMED |
| `:187` (task 11 RangeError) | Line 187: `data.sublist(offset, offset + contentLen)` — throws `RangeError` on truncated input. | CONFIRMED |

### Test files — structural markers

| tasks.md citation | Verified content | Status |
|-------------------|-----------------|--------|
| `hpke_test.dart` "after line 200" (task 14 seq overflow) | Line 200 is inside the `seal seq=1` test (the `await ctx.seal(aad1, ...)` call). The `seal seq=1` test ends at line 200. A new `seal seq overflow` test would be inserted after the existing seq tests, which end around line 200. File has 297 lines total; there is room. | CONFIRMED — marker is accurate |
| `hpke_test.dart` "after line 241" (task 33 export invalid length) | Line 241 is the last `});` of the `export with "TestContext"` test. The `HPKE RFC 9180` group closes at line 242. The `HKDF RFC 5869` group starts at line 244. An export-invalid-length test would go between 242 and 244 or at end of file. | CONFIRMED — marker is accurate |
| `hpke_test.dart` (task 15 invalid public key) | No existing test for `setupBaseS` with an invalid/all-zero public key. File has 297 lines. | CONFIRMED — gap exists |
| `bhttp_test.dart` "after line 68" (task 16 8-byte varint) | Line 68 is `});` closing the `varint roundtrip` group. The `serializeRequest` group starts at line 71. An 8-byte boundary test would slot between 68 and 71. | CONFIRMED — marker is accurate |
| `bhttp_test.dart` "after line 185" (task 17 truncated body) | Line 185 is `});` closing the `rejects non-response framing` test. Line 186 is `}` closing the `parseResponse` group. A truncated-body test would go inside the `parseResponse` group (before line 186) or after it. | CONFIRMED — marker is accurate |
| `ohttp_test.dart` "after line 62" (task 20 multi-suite KeyConfig) | Line 62 is blank. Line 63 is `}` closing the `OhttpKeyConfig.parse` group. The `OhttpKeyConfig.validate` group starts at line 65. A multi-suite test would go at the end of the parse group or as a new group after line 63. | CONFIRMED — marker is accurate |
| `ohttp_test.dart` "after line 163" (task 19 AEAD auth failure) | Line 163 is `});` closing the second `ohttpDecapsulate` test. Line 164 is `}` closing the `ohttpDecapsulate` group. File has 165 lines. An AEAD auth failure test would be appended after line 163. | CONFIRMED — marker is accurate |

### Stale Citations

No citations were found to be incorrect. Every line-number reference in tasks.md §2 matches the current
source as of the branch `feature/AW-2865-investigation-ohttp_dart`. The one nuance worth noting in task
files:

- **Task 02 (enforce-https-scheme)** cites `:43–51`. The `OhttpGatewayConfig` constructor spans lines
  43–50 (closing `);` at 50). Line 51 is blank. Line 52 is `effectiveDirectBaseUrl`. The task file
  should cite `:43–50, 52` for complete coverage, but `:43–51` is not misleading — the range unambiguously
  includes the constructor. Add `:52` to the Files field in the task file for completeness.

- **Task 01 (ohttp-bypass-send-direct)** cites `:52, 148–171`. Line 147 is the doc comment for
  `sendDirect()`; line 148 starts the method body. The citation `:148–171` correctly covers the full
  method body. Including line 147 in the task file is optional but useful for showing the missing
  `@Deprecated` annotation.

---

## Task 32 Severity Escalation Recommendation

**Original classification:** IMPROVEMENT
**Recommendation:** Escalate to **HIGH**

### Evidence

1. `effectiveDirectBaseUrl` is a **public getter** on `OhttpGatewayConfig` (line 52 of
   `lib/src/ohttp_client.dart`). Because it is public, any caller — including test harnesses, logging
   utilities, and debugging code in downstream wallet apps — can read it and observe that the value is
   `gatewayBaseUrl` when `directBaseUrl` is null. This makes the silent fallback visible to code that
   was not intended to handle it, widening the blast radius of the task 01 footgun.

2. The getter is the only mechanism through which `sendDirect()` resolves its target URL (line 154:
   `Uri.parse('${gateway.effectiveDirectBaseUrl}$path')`). Any developer who discovers the getter via
   IDE autocomplete and calls it to "check what URL will be used" receives a silently incorrect answer:
   they see `gatewayBaseUrl` and may conclude that `sendDirect()` targets the gateway (the relay), not
   a direct target — reinforcing the confusion rather than correcting it.

3. Task 01 (BLOCKER) proposes adding a runtime warning / `@Deprecated` annotation on `sendDirect()`.
   Task 32 is described in tasks.md §2 as "companion, not duplicate — P6-1 adds a runtime warning; this
   changes API surface." Changing API surface (making the getter private or removing it) is a distinct
   and complementary mitigation: it eliminates the incorrect fallback behaviour from the public API
   entirely, rather than warning about it after the fact. For a wallet context where privacy
   unlinkability is a first-class security property, reducing the public API footprint that exposes this
   footgun warrants HIGH rather than IMPROVEMENT.

4. The risk is not speculative: `OhttpGatewayConfig` is part of the public API re-exported by
   `lib/ohttp_dart.dart` (via `lib/src/ohttp_client.dart`). The getter requires no imports beyond the
   standard client instantiation, so it is immediately accessible to all library consumers.

### Reasoning against BLOCKER

The getter itself does not cause any privacy breach in isolation — it only returns a value. The breach
occurs only if a caller passes that value to an HTTP request without OHTTP wrapping. Task 01 (which is
BLOCKER) addresses the primary action (`sendDirect()`). Task 32 addresses the API surface that enables
discovery of the footgun. HIGH is the appropriate level for an API-surface issue that amplifies a
BLOCKER.

### Required in-task documentation

Per PRD constraint 8, the task 32 file must document:
- Original classification: IMPROVEMENT
- New classification: HIGH
- Evidence: public getter enables direct URL observation; companion to BLOCKER task 01 by widening the
  footgun's discoverable surface.

---

## Patterns Used

- **No scheme validation pattern.** `OhttpGatewayConfig` accepts `gatewayBaseUrl` as a plain `String`
  with no `startsWith('https://')` check. The same pattern applies to `directBaseUrl`. Task 02 must
  propose validation at construction time.

- **No timeout / no cache pattern.** `OhttpClient` has only two fields: `_httpClient` and `gateway`.
  There is no TTL field, no `OhttpKeyConfig?` cache field. Every `send()` call fetches the KeyConfig
  fresh (two round trips per inner request). Tasks 08 and 09 address these gaps.

- **Unguarded `RangeError` pattern.** `decodeVarint` and all `data.sublist(...)` calls in `bhttp.dart`
  let Dart throw `RangeError` (an `Error`, not `Exception`) on truncated input. No `_requireBytes`
  helper exists. Tasks 11 addresses this.

- **Mixed exception hierarchy pattern.** `OhttpKeyConfig.parse()` throws `FormatException` for
  unsupported KEM; `OhttpKeyConfig.validate()` throws `UnsupportedError` for the same conceptual
  condition. `ohttpDecapsulate` surfaces `SecretBoxAuthenticationError` from `package:cryptography`
  directly. Tasks 07 and 23 address these.

- **Plain HKDF vs. labeled HKDF split.** HPKE key schedule uses `_labeledExtract` / `_labeledExpand`
  (hpke.dart). Response decapsulation uses unlabeled `HpkeSender.hkdfExtract` / `hkdfExpand`
  (ohttp.dart lines 193, 196, 203). These are intentionally different but are not distinguished by a
  maintenance comment. Task 29 addresses this.

- **Retained key material pattern.** `HpkeSenderContext` stores `key`, `baseNonce`, `exporterSecret`
  as `final Uint8List` fields with no zeroization. `OhttpEncapsulateResult` similarly holds
  `exportedSecret`. Tasks 04 and 05 address this.

- **onLog callback pattern.** `OhttpClient.send()` accepts an optional `void Function(String)?`
  `onLog` parameter (line 74). The callback currently logs URL fragments including
  `gatewayBaseUrl + configPath` (line 77) and `gatewayBaseUrl + requestPath` (line 113). This is a
  partial observability mechanism — it is opt-in but logs sensitive path information that tasks.md §4
  identifies as restricted. Task 13 must address the logging constraints.

---

## Limitations & Risks

1. **No integration tests exist.** `test/integration/` does not exist. All three test files test only
   happy-path or simple error conditions against in-process mocked or synthetic data. Task 21 proposes
   creating this directory and a gateway-stub test; the plan-phase engineer will need to choose a stub
   approach (e.g., `package:shelf` local server).

2. **Single fixed cipher suite.** The library has no cipher suite negotiation. All KEM/KDF/AEAD
   constants are hard-coded in `hpke.dart`. Any gateway that advertises a different primary suite will
   cause task 06's silent-drop problem to surface as a runtime failure with an unhelpful error.

3. **Public API boundary is enforced only by convention.** `lib/src/` is internal by convention;
   there is no `package:meta` `@internal` enforcement. Task files citing internal symbols must note
   that the fix involves internal-only changes unless the affected symbol is also re-exported.

4. **`effectiveDirectBaseUrl` is publicly accessible.** As noted in the severity escalation analysis,
   this getter widens the footgun surface of task 01. Making it private is a breaking change (semver
   major) if any consumer currently references it; however, since `publish_to: none` is set, the
   package is not published, so breaking-change concerns are scoped to internal consumers only.

5. **`onLog` leaks path information.** The existing `onLog` callback in `send()` logs
   `gatewayBaseUrl + configPath` and `gatewayBaseUrl + requestPath` as formatted strings. A caller who
   routes these log strings to a remote logging service may inadvertently expose the gateway URL and
   path structure. Task 13 must include a constraint that the `onLog` callback should never surface
   the inner request path, `targetAuthority`, or key material.

6. **`AesGcm.decrypt` exception leaks transitive dependency.** `SecretBoxAuthenticationError` from
   `package:cryptography` is not re-exported by `lib/ohttp_dart.dart`. Any downstream code that tries
   to catch authentication failures must add a direct `package:cryptography` import, making the
   transitive dependency part of the public contract by necessity. Task 07 must propose a library-owned
   `OhttpDecryptionException` wrapper.

---

## New Technical Questions

The following questions surfaced during research and may be relevant to the plan/implementation phase
that consumes the task files:

1. **Task 04 (zeroize HPKE key material):** Dart does not provide a native secure-zeroize API for
   `Uint8List`. The implementation will need to manually fill the list with zeros before releasing the
   reference (`key.fillRange(0, key.length, 0)`). The task file should note that this is a best-effort
   mitigation (the JIT/AOT compiler may not guarantee the write is not optimized away) and propose
   evaluating whether `package:cryptography` surfaces a `SecretKey.destroy()` equivalent that provides
   stronger guarantees.

2. **Task 13 (structured observability hooks) + §4 logging constraints:** The existing `onLog`
   callback uses a plain `String` parameter. A structured hook would use a structured event object.
   The task file should specify whether the proposed hook is additive (new parameter/callback) or
   replaces `onLog`. Since `onLog` is already part of the public API (`send()` signature), replacing
   it is a breaking change.

3. **Task 21 (gateway stub integration test):** The tasks.md citation says `new test/integration/`
   directory. No existing scaffolding is present. The task file should note that a real or mock gateway
   is needed and propose using `package:shelf` or a recorded HTTP fixture to avoid network
   dependencies in CI.

4. **Task 32 file naming after escalation:** The slug in tasks.md §2 is
   `make-effective-direct-base-url-private`. The task file number is 32 (in the IMPROVEMENT sequence,
   now proposed as HIGH). Per the user's confirmed numbering rule (no renumbering), the file remains
   `32-make-effective-direct-base-url-private.md` even after the severity escalation.
