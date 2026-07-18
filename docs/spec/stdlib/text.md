# Spec — `std.text` / `string` (UTF-8 strings: construction, slicing, parsing, encoding)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-text` (M-524, #165, Batch P5-B; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.text` / `string` · Ring `2` (RFC-0016 §4.2) · Tier `B` (RFC-0016 §4.4) |
| **Tracks** | `M-524` (#165) — the Phase-5 task this spec delivers (RFC-0016 §4.4 `text`/`string` row) |
| **Scope** | The UTF-8 string **type** and its operations: construction, slicing/indexing on validated char/grapheme boundaries, the parse surface (`str → T` as `Result`), and encoding/transcoding (UTF-8 ⇄ UTF-16/Latin-1/bytes). Value-semantic, immutable by default. |
| **Boundary** | Out of scope: the dual human/machine **projection** of *other* values to text (`display`/`debug`/`to_json`) — that is `fmt` (M-533); `text` owns the string TYPE and `parse`/`slice`/`encode`, `fmt` owns projection-of-values-to-text. Ordering/equality of strings is `cmp` (M-532). Serializing a string to bytes for IO / the wire is `io`/`serialize` (M-514). The numeric VALUE semantics of a parsed number live in `math`/`numerics` (M-525/M-512); `text` owns only the **parse-failure surface** (the byte/char span + expected-vs-found). |
| **Depends on** | RFC-0016 §4.1 (the C1–C6 contract) and §4.4 (the `text` row); RFC-0001 (the value model — `Value`/`Repr`/`Meta`, the guarantee lattice §4.3, content-addressing §4.6); RFC-0013 §4.1 (the structured diagnostic record / I1 — the parse-failure trace); ADR-003 (content-addressed identity; metadata is not identity). |
| **Grounds on** | Ring-0 `core` (M-515 — `Option`/`Result`/error values, the lattice tags) and the kernel value model; `fmt` (M-533) for the inverse projection direction. KC-3: `text` adds no trusted code — it is a value-model and `core` **consumer**. |

---

## 1. Summary

`std.text` is the UTF-8 string surface every program needs: a value-semantic, immutable `Text` type with
construction from bytes/chars, slicing and indexing on **validated** char/grapheme boundaries, a parse
family (`str → T`), and encoding/transcoding between UTF-8 and other encodings. Its **honesty crux** is
twofold and structural, exactly as RFC-0016 §4.4 names it for `text`: (1) `parse` returns a **`Result`,
never a sentinel** — a failed parse is `Err` carrying *where* it failed (the byte/char index) and
*expected-vs-found*, never `0`/`""`/a best-effort value (C1/G2); and (2) a **lossy encoding/transcoding is
an explicit `Err`, never a silent replacement-character (U+FFFD) substitution** — a caller who *wants*
lossy behaviour must reach for a **distinct, named** `*_lossy` op that returns the substitution count, so
the lossiness is in the type, not a hidden default. Almost every op is `Exact` (string operations carry no
accuracy/precision semantics); a `parse` carries no false precision — it is `Exact`-when-`Ok`. Ring 2,
Tier B; it adds **no trusted code** (KC-3), consuming the kernel value model and Ring-0 `core`.

## 2. Scope & module boundary

- **In scope:**
  - **Construction:** `Text` from a `&str` literal, from a `[byte]` (validated → `Result`), from a
    `[char]` (total); concatenation/`join`; immutable transforms (`to_upper`/`to_lower`/`trim`/`replace`)
    that return a **new** `Text` (value-semantic, never in-place).
  - **Slicing / indexing:** sub-ranges and indexing on **validated** char/grapheme-cluster boundaries; an
    out-of-range or mid-codepoint/mid-grapheme split is an explicit `Err`, never a silent truncation to a
    nearest boundary.
  - **Parse:** `parse::<T>(s) -> Result<T, ParseErr>` for the primitive `T` the stdlib defines (`Int`,
    `Bool`, …) — `text` owns the **parse-failure surface** (the span + expected/found), routing the parsed
    *value* semantics to the owning module.
  - **Encoding / transcoding:** UTF-8 ⇄ UTF-16/Latin-1/bytes; a lossy direction is an explicit `Err` plus a
    **distinct named** opt-in `*_lossy` variant.
- **Out of scope (and who owns it):**
  - **Projection of *other* values to text** (`display`/`debug`/`to_json`/`from_json`) — `fmt` (**M-533**).
    `text` is the string *type and operations*; `fmt` is the *render-a-`Value`-as-text* projection. The
    seam: `fmt` produces a `Text`; `text` consumes/produces `Text`. `text` does **not** define `display`.
  - **Ordering / equality** of strings (`eq`, `cmp`, collation) — `cmp` (**M-532**). `text` exposes no
    `cmp`/`ord`; a sorted-string need routes through `cmp`'s trait surface.
  - **Serializing a `Text` to wire bytes / single-consumption IO** — `io`/`serialize` (**M-514**). `text`'s
    `encode` produces an in-memory `[byte]` value; *writing* those bytes to a substrate is M-514's affine,
    single-consumption surface (LR-8).
  - **The numeric VALUE semantics of a parsed number** (range, rounding, the honest ε if any) —
    `math`/`numerics` (**M-525 / M-512**). `text` owns the *parse-failure* surface; the parsed value's tag
    and bounds are the numeric module's. (Seam FLAGGED §7-Q3.)
  - **A representation change** (binary↔ternary, dense↔VSA) — `swap` (**M-516**). Encoding is **not** a
    swap: it changes a byte *encoding*, not a `Repr` paradigm, and emits no `SwapCertificate`.
- **Ring & layering:** Ring 2 (RFC-0016 §4.2). `text` is **new library code written to the §4.1 contract
  over Ring 0** (the `Value`/`Repr`/`Meta` re-exports and `Option`/`Result` from `core`, M-515). It
  *wraps* nothing trusted and *builds new* the string type + its ops; it never enlarges the trusted base
  (KC-3) and asserts no `wild`/FFI (pure value logic; FLAGGED §7-Q4 for the validation floor).

## 3. Exported-op surface (design sketch)

A signature sketch — value-semantic, immutable-by-default. Construction/slicing that can land off a valid
boundary returns `Result`; total ops return their value directly; effect-free throughout. The parse and the
lossy-encode arms carry their explicit error / lossiness in the **type**. This is a DESIGN sketch to fix the
surface and feed the matrix, not a committed grammar.

```
// illustrative signatures (not a committed surface)

// --- construction (value-semantic; transforms return a NEW Text) ---
from_chars(cs: [Char]) -> Text                          // total: every char sequence is valid UTF-8
from_utf8(bytes: [Byte]) -> Result<Text, Utf8Error>     // fallible: invalid byte sequence -> Err(at index)
concat(a: &Text, b: &Text) -> Text                      // total
join(parts: [Text], sep: &Text) -> Text                 // total
to_upper(s: &Text) -> Text                              // total; NEW value, never in-place
trim(s: &Text) -> Text                                  // total
replace(s: &Text, from: &Text, to: &Text) -> Text       // total

// --- length / iteration (no accuracy semantics: Exact) ---
len_bytes(s: &Text)     -> Count                         // total
len_chars(s: &Text)     -> Count                         // total
len_graphemes(s: &Text) -> Count                         // total (grapheme segmentation; basis FLAGGED §7-Q2)
chars(s: &Text)         -> Iter<Char>                    // total (lazy; see iter, M-526)

// --- slicing / indexing on VALIDATED boundaries (C1: off-boundary is Err, never a silent snap) ---
slice(s: &Text, range: ByteRange) -> Result<Text, BoundaryError>
//   Err(OutOfRange { len, asked }) | Err(NotCharBoundary { byte }) | Err(NotGraphemeBoundary { byte })
char_at(s: &Text, i: CharIndex)    -> Result<Char, BoundaryError>   // Err(OutOfRange) — never a sentinel char

// --- parse: str -> T is a Result, NEVER a sentinel (the honesty crux) ---
parse_int(s: &Text)  -> Result<Int,  ParseErr>          // Err carries WHERE + expected-vs-found
parse_bool(s: &Text) -> Result<Bool, ParseErr>
parse<T>(s: &Text)   -> Result<T, ParseErr>             // generic; value semantics owned by T's module

// --- encoding / transcoding: a lossy direction is an explicit Err, NEVER silent U+FFFD ---
encode_utf8(s: &Text)        -> [Byte]                          // total: Text is already UTF-8
to_utf16(s: &Text)           -> [U16]                           // total: UTF-8 -> UTF-16 is lossless
to_latin1(s: &Text)          -> Result<[Byte], EncodeError>     // Err(Unrepresentable { char, at }) on a non-Latin-1 char
to_latin1_lossy(s: &Text)    -> Lossy<[Byte]>                   // DISTINCT, NAMED opt-in: { bytes, substituted: Count }
from_utf16(units: [U16])     -> Result<Text, TranscodeError>    // Err(unpaired surrogate at index) — never silent U+FFFD

// --- the explicit error sets (never a sentinel; each carries a WHERE) ---
enum Utf8Error      { Invalid { byte: Index, reason } }
enum BoundaryError  { OutOfRange { len, asked }, NotCharBoundary { byte }, NotGraphemeBoundary { byte } }
enum ParseErr       { Empty, Invalid { at: Index, expected, found }, OutOfRange { at: Index, target } }
enum EncodeError    { Unrepresentable { ch: Char, at: Index, target_encoding } }
enum TranscodeError { UnpairedSurrogate { at: Index }, Invalid { at: Index } }

// Lossy<T> is the TYPE-LEVEL opt-in: a value PLUS the substitution count, so lossiness is never hidden.
struct Lossy<T> { value: T, substituted: Count, marker: Char }   // marker = the requested replacement char
```

The **`Result` vs `Lossy` split is the honesty crux made type-level**: the default transcode arm is
`Result` (a lossy input is `Err`, naming the offending char and its index — C1/G2), and the *only* way to
get U+FFFD substitution is to call the **distinct, named** `*_lossy` op, whose return type is `Lossy<T>`
carrying the substitution count. A caller cannot lose data silently: either they handle the `Err`, or they
explicitly opted into `Lossy` and receive the count of what was substituted.

## 4. Guarantee matrix (the load-bearing deliverable — RFC-0016 §4.5)

Rows = exported ops (grouped: construct/slice/index · format-adjacent · parse · encode/transcode). To be
encoded as a checked table (the RFC-0003 §4 template) and asserted in tests once code lands — never prose
only. **Every row is `Exact`**: `text` carries no accuracy/precision/probability semantics (a string op
either returns the right characters/bytes or an explicit `Err`), so VR-5/C2 makes each op `Exact` rather
than a downgraded tag — the honesty load is carried entirely by the **fallibility column** and the
parse/transcode **trace**.

| Op | Guarantee tag | Fallibility (explicit error set) | Declared effects | EXPLAIN-able? |
|---|---|---|---|---|
| `from_chars` / `concat` / `join` | `Exact` | total | `none` | n/a |
| `to_upper` / `to_lower` / `trim` / `replace` | `Exact` | total (returns a NEW value; never in-place) | `none` | n/a |
| `from_utf8` | `Exact` *(when `Ok`)* | `Err(Utf8Error::Invalid { byte, reason })` — **never a silent U+FFFD repair** | `none` | yes (the error names the invalid byte) |
| `len_bytes` / `len_chars` / `len_graphemes` | `Exact` | total | `none` | n/a |
| `chars` (lazy iter) | `Exact` | total | `none` | n/a |
| `slice` (validated boundary) | `Exact` *(when `Ok`)* | `Err(OutOfRange \| NotCharBoundary \| NotGraphemeBoundary)` — **never a silent snap/truncation** | `none` | yes (the error names the offending byte index) |
| `char_at` | `Exact` *(when `Ok`)* | `Err(OutOfRange)` — **never a sentinel char** | `none` | yes (index + length) |
| `parse_int` / `parse_bool` / `parse<T>` | `Exact` *(when `Ok`)* | `Err(ParseErr::{ Empty, Invalid{at,expected,found}, OutOfRange{at,target} })` — **never a sentinel `0`/`false`** | `none` | yes (the diagnostic carries WHERE + expected-vs-found, RFC-0013 §4.1) |
| `encode_utf8` / `to_utf16` | `Exact` | total (lossless: UTF-8 → UTF-8/UTF-16) | `none` | n/a |
| `to_latin1` (default, strict) | `Exact` *(when `Ok`)* | `Err(EncodeError::Unrepresentable { ch, at })` — **never a silent U+FFFD** | `none` | yes (names the unrepresentable char + index) |
| `to_latin1_lossy` (DISTINCT named opt-in) | `Exact` | total — returns `Lossy{ value, substituted, marker }`; the substitution count is **in the value** | `none` | yes (the `Lossy` record reifies how many chars were substituted + the marker) |
| `from_utf16` | `Exact` *(when `Ok`)* | `Err(TranscodeError::{ UnpairedSurrogate{at}, Invalid{at} })` — **never silent U+FFFD** | `none` | yes (names the offending unit index) |

**Tag justification (VR-5 — downgrade rather than overclaim is moot; nothing reaches above `Exact`):**
- **All rows are `Exact`.** No `text` op carries accuracy/precision/probability semantics — there is no ε
  bound, no probability, no approximation. A length, a slice, a parse, an encode either yields the exact
  bytes/chars/value or an explicit `Err` (RFC-0016 §4.1 C2: "an op with no accuracy semantics … is simply
  `Exact`"). A `parse` is `Exact`-when-`Ok` and `Err`-when-not: it carries **no false precision** — there
  is no "approximately parsed" state. The honesty here is **not** a guarantee tag but the **fallibility
  column**: every restricted op returns its explicit error set, never a sentinel.
- **No `Proven`/`Empirical`/`Declared` anywhere.** No op claims a bound, so no theorem side-condition is
  asserted and the downgrade discipline does not bite. The single place a *future* op might carry a bound
  is grapheme segmentation against a versioned Unicode table (a `Declared`-against-a-named-table concern),
  FLAGGED §7-Q2 rather than asserted.
- **No silent replacement / truncation / sentinel anywhere** (C1/G2): `from_utf8`, `to_latin1`,
  `from_utf16` refuse a lossy/invalid input with an explicit `Err` naming **where**; `slice`/`char_at`
  refuse an off-boundary/out-of-range index with an explicit `Err`; `parse_*` refuse a malformed input
  with an explicit `Err` carrying the index + expected-vs-found. The **only** path to U+FFFD substitution
  is the distinct, named `*_lossy` op whose `Lossy` return type reifies the substitution count. The
  explicit error / typed lossiness *is* the never-silent guarantee.

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2):** This is the module's whole reason for being. `parse_*` returns `Result`,
  **never** a sentinel (`0`/`false`/`""`) — a malformed input is `Err(ParseErr)` carrying the byte/char
  index and expected-vs-found (the RFC-0013 §4.1 diagnostic record). A lossy encoding/transcoding
  (`from_utf8`, `to_latin1`, `from_utf16`) is an explicit `Err` naming the offending char/byte and its
  index — **never** a silent U+FFFD replacement-character substitution; the **only** way to get U+FFFD is
  the **distinct, named** `*_lossy` op whose `Lossy{substituted}` return type makes the lossiness
  un-droppable. A `slice`/`char_at` off a valid boundary or out of range is `Err(BoundaryError)`, **never**
  a silent snap to the nearest boundary or a truncation. No op returns a sentinel, a clamp, or a partial
  result.
- **C2 — honest per-op tag (VR-5):** Every op is `Exact` and the matrix says so — `text` has no
  accuracy/precision semantics, so there is nothing to downgrade and nothing claims `Proven`/`Empirical`/
  `Declared`. A `parse` is honestly `Exact`-when-`Ok`: it carries **no false precision** (no "approximately
  parsed" tag), because either the input denotes the value exactly or it is an explicit `Err`. The honesty
  load is the fallibility column, not a probabilistic tag.
- **C3 — no black boxes / EXPLAIN (SC-3/G11):** Every op that can *reject* reifies *why* in an inspectable
  diagnostic — a `ParseErr` carries the index + expected-vs-found, a `BoundaryError` carries the offending
  byte index and the length, a `Utf8Error`/`TranscodeError` names the invalid unit, an `EncodeError` names
  the unrepresentable char. These are the EXPLAIN-able artifacts (RFC-0013 §4.1: additive, the explicit
  error still propagates, I1). The `*_lossy` op reifies its substitution count in the `Lossy` record — the
  EXPLAIN-able artifact of *what* it lost. Total, unrestricted ops (`concat`, `len_*`, `to_upper`) neither
  select, convert, nor approximate, so they carry no EXPLAIN obligation (EXPLAIN is `n/a`, not
  absent-and-needed).
- **C4 — content-addressed, value-semantic (ADR-003 / RFC-0001):** A `Text` is an **immutable value**;
  every transform (`to_upper`, `trim`, `replace`, `slice`) returns a **new** `Text` and never mutates its
  input — there is no in-place string mutation in the surface. Two `Text` values with equal content are the
  same value (content-addressed identity, RFC-0001 §4.6); metadata (a source span, a name) is **not**
  identity (ADR-003). All ops are pure functions of their inputs.
- **C5 — above the small kernel (KC-3):** Ring 2. `text` consumes the kernel value model and Ring-0 `core`
  (M-515) re-exports and adds **no** trusted code — it produces ordinary values, never a certificate, so it
  adds nothing the certificate checker must trust. UTF-8 validation, boundary checks, and parsing are pure
  value logic; **no `wild`/FFI is asserted at the `text` layer** (the validation/segmentation compute floor
  is FLAGGED §7-Q4 — if grapheme segmentation bottoms out in an FFI Unicode library, that would inherit a
  `wild` block to inventory, LR-9, narrowing this claim).
- **C6 — declared, bounded effects (RFC-0014):** Every `text` op is **pure** — `effects: none` across the
  matrix. No IO, no clock, no randomness, no global state. (Writing encoded bytes to a substrate is `io`'s
  effectful, single-consumption surface — M-514 — not a `text` op; `encode_utf8` only *produces* an
  in-memory byte value.)

## 6. Grounding

- The **contract** (C1–C6) and the `text`/`string` row (the honesty crux "`parse` is `Result`, never a
  sentinel; lossy encoding is an explicit error"): RFC-0016 §4.1 and §4.4. The guarantee-matrix obligation:
  RFC-0016 §4.5 (the RFC-0003 §4 template).
- The **value model** (`Value`/`Repr`/`Meta`, `Option`/`Result`, the guarantee lattice §4.3,
  content-addressing §4.6): RFC-0001, via Ring-0 `core` (M-515).
- The **parse-failure trace** (the byte/char index + expected-vs-found, additive and still-propagating,
  I1): RFC-0013 §4.1 (the structured diagnostic record; "a diagnostic is additive over an explicit error
  and never replaces it").
- **Immutability / value-semantics / metadata-is-not-identity:** ADR-003 (content-addressing; Foundation
  §5.1 — "formatting is a *projection*, not a mutation of identity") and RFC-0001 §4.6, via clause C4.
- **Never-silent:** G2 (the foundational never-silent guarantee) via C1 — the `parse`-`Result` and the
  strict-vs-`*_lossy` encoding split are the structural form of G2 for text.
- **Boundaries:** `fmt` (M-533, the value→text projection), `cmp` (M-532, string ordering/equality),
  `io`/`serialize` (M-514, bytes→wire), `math`/`numerics` (M-525/M-512, parsed-value semantics), `swap`
  (M-516, representation change) — all per RFC-0016 §4.3/§4.4 and the index [`README.md`](./README.md).
  The structural template is [`_TEMPLATE.md`](./_TEMPLATE.md).

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) The `Lossy<T>` opt-in shape, and whether a single replacement char is configurable.** The sketch
  makes lossy transcoding a **distinct named** op returning `Lossy{ value, substituted, marker }`. Is the
  replacement `marker` fixed at U+FFFD, or a caller-supplied char (still type-level opt-in)? And is `Lossy`
  a distinct type or a `Meta`-attached substitution count on the byte value? — *Disposition: FLAGGED;
  proposed default is a distinct `Lossy` type with a U+FFFD-default-but-overridable marker, so the lossiness
  is always in the type. Ties to RFC-0016 §8-Q3 (ergonomics vs the contract).*
- **(Q2) Grapheme-cluster segmentation and its Unicode-table basis.** `len_graphemes` /
  `NotGraphemeBoundary` depend on a Unicode grapheme-break table whose version is a real, versioned
  dependency. A segmentation result is `Exact` *against a named table version* — but which table, and
  whether the version is reified in `Meta` (so a cross-version difference is not silent), is unsettled. —
  *Disposition: FLAGGED; the table version must be a reified, inspectable input (C3), not a hidden default —
  the exact table/version is a maintainer call. Whether char-boundary slicing (codepoint-level) ships before
  grapheme-level is a sequencing sub-question.*
- **(Q3) The `parse` ↔ `math`/`numerics` value-semantics seam.** `text` owns the **parse-failure surface**
  (the span + expected-vs-found) and routes the parsed *value* (its range, its tag, any honest bound) to
  the owning module (`math`/`numerics`, M-525/M-512). The exact signature of how `parse_int` hands off an
  out-of-range case (`ParseErr::OutOfRange` here vs a `math`/`convert` narrowing error) needs maintainer
  sign-off so the failure is not double-owned. — *Disposition: FLAGGED; proposed default is that `text` owns
  the *lexical* failure (malformed digits, span) and the numeric module owns the *value-range* failure, with
  `parse` returning the former and delegating the latter. Coordinate with M-525/M-532 (the `str → i32`
  parse-vs-convert seam the `cmp` spec also names, cmp §7-Q2).*
- **(Q4) The UTF-8-validation / segmentation compute floor — `wild`/FFI?** §5 asserts no `wild` at the
  `text` layer. If grapheme segmentation or a fast UTF-8 validator bottoms out in a platform/FFI Unicode
  library (rather than a pure trusted routine), `text` inherits a `wild` block to inventory (LR-9), and the
  C5 "no `wild`" claim narrows. — *Disposition: FLAGGED; default to a pure routine; ties to RFC-0016 §8-Q6
  (the `wild`/FFI floor / `std-sys` split), the same question `math`'s transcendental floor raises.*

## Meta — changelog

- **2026-06-17 — Draft (needs-design).** Stands up the `std.text` / `string` (M-524, #165) module spec
  under **RFC-0016 (Draft)**: Ring 2 / Tier B UTF-8 strings — construction, validated char/grapheme
  slicing/indexing, the parse family, and encoding/transcoding, value-semantic and immutable by default.
  The **honesty crux** is twofold and structural: (1) `parse` returns a **`Result`, never a sentinel** — a
  failure is `Err(ParseErr)` carrying the byte/char index + expected-vs-found (the RFC-0013 §4.1 diagnostic,
  I1); (2) a **lossy encoding/transcoding is an explicit `Err`, never a silent U+FFFD** substitution — the
  only path to replacement is a **distinct, named** `*_lossy` op whose `Lossy{substituted}` return type
  makes the lossiness un-droppable (C1/G2). The boundary is fixed against `fmt` (M-533, the value→text
  projection — `text` owns the string TYPE + parse/slice/encode, `fmt` owns projection-of-values-to-text),
  `cmp` (M-532, ordering/equality), `io`/`serialize` (M-514, bytes→wire), and `math`/`numerics`
  (M-525/M-512, parsed-value semantics). The load-bearing **guarantee matrix** (RFC-0016 §4.5) carries
  twelve rows, **all `Exact`** (no accuracy semantics — the honesty is in the fallibility column), with
  every restricted op (`from_utf8`, `slice`, `char_at`, `parse_*`, `to_latin1`, `from_utf16`) returning an
  explicit error set that names **where** it failed, never a sentinel/snap/silent-replacement. §4.1
  conformance (C1–C6) stated per clause; grounding traces to RFC-0016 §4.1/§4.4/§4.5, RFC-0001 (value model,
  §4.6 content-addressing), RFC-0013 §4.1 (the parse trace), ADR-003 (metadata-is-not-identity), G2/VR-5/
  KC-3. Four questions FLAGGED (the `Lossy` opt-in shape, grapheme-table basis/versioning, the
  parse↔numerics value-semantics seam, the UTF-8/segmentation `wild`/FFI floor), tied to RFC-0016
  §8-Q3/§8-Q6. No code; no kernel change (KC-3, Ring 2, no trusted-base growth). Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).

- **2026-06-27 — Self-hosted UTF-8 validity prototype now executes three-way (M-717, rsm S2).** A *distinct artifact* from this Rust-first spec: `lib/std/text.myc` (RFC-0031 §5 D4) is a self-hosted Mycelium-lang prototype of the **validity layer** — the byte-classification gates `reject_two`/`reject_three`/`reject_four` over 32-bit binary thresholds, refining the parse-failure into a **4-variant** `Utf8Error = Invalid(Binary{8}) | Overlong(Binary{8}) | Surrogate(Binary{8}) | TooLarge(Binary{8})` (this spec's §3 sketch carries a single `Invalid` variant; the self-hosted prototype distinguishes the four honest rejection causes, each naming the offending byte). It executes three-way (L1-eval ≡ L0-interp ≡ AOT; `crates/mycelium-l1/tests/std_text.rs`), with every rejection an explicit never-silent `Err` — **never** a U+FFFD repair (C1/G2). It is a **subset** of this spec's surface (validity classification only — no slicing/parse/transcode) and does **not** change this spec's status; the full Mycelium-lang migration (M-502-gated) still remains. Agreement `Empirical`. Append-only; no kernel change (KC-3).
