# 0001 — Remote Session Control: Attach & Take-Control for newt-agent over agent-mesh

- Status: Draft — proposed, not yet approved
- Date: 2026-07-02
- Repos affected: `agent-mesh`, `newt-agent`, `agent-bridle`, `newt-remote` (this repo, new)

## 1. Summary

Let a human join a `newt-agent` session that is already running somewhere
else — a `tmux` pane over SSH on `gnuc`, a box behind the home WireGuard
tunnel, a laptop on the LAN — from another device, without starting a
second agent process. Two levels of joining:

- **Attach**: watch the session's output live, read-only.
- **Control**: take over keyboard input, explicitly and revocably.

The first client is an Android app. Since `newt-agent`'s RichTUI already
renders Markdown to a stream of complete, styled blocks before it ever
touches a terminal, the Android app doesn't emulate a terminal — it
consumes that same block stream and renders it natively, with its own
widgets for buttons/choices instead of ANSI text.

Identity, authorization, and transport are **not new** — they're
`agent-mesh`: ed25519 identity cross-signed by the user's existing GitHub
SSH key, authenticated QUIC transport, and pub/sub/request-reply
messaging. This proposal is mostly about *using* agent-mesh's existing
capability-delegation machinery for a new purpose, plus one genuinely new
piece: a capability axis that expresses *how strongly* a control claim was
authenticated (bare possession of a key vs. a locally-verified human).

## 2. Goals

1. Attach to a live `newt-agent` session from another device, read-only,
   by default.
2. Explicitly take control (write input) from another device, gated by a
   capability distinct from the attach capability.
3. Reuse `agent-mesh`'s existing identity/delegation/transport — add the
   minimum necessary net-new primitives, don't build a parallel auth
   system.
4. Work over the home WireGuard tunnel and Tailscale, not just bare LAN.
5. Give the Android client clean, structured events (text blocks, choice
   prompts, text-input prompts) — not a terminal emulator.
6. Be honest in the UI when a control grant is weaker than intended (e.g.
   no biometric factor available on the requesting device) rather than
   silently upgrading trust.

## 3. Non-goals

- Multi-viewer conflict resolution beyond single-writer-at-a-time
  ("explicit take control" replaces whoever currently holds Control; it
  does not merge concurrent input).
- Session recording/replay, or any UI beyond a single active session.
- Changing how `newt-agent` behaves when there is no remote viewer at
  all — the local TUI experience is unaffected.
- Building a general-purpose remote-desktop or terminal-sharing product.
  This is scoped to newt-agent sessions specifically.
- iOS. Android first; revisit once the protocol is stable.

## 4. Background: what already exists

Grounded against the current state of each repo (2026-07-02):

**agent-mesh** (`Gilamonster-Foundation/agent-mesh`, published to crates.io,
currently 0.6.x):
- `agent-mesh-protocol`: `UserKey` (ed25519 root, cross-signed by the
  user's GitHub SSH key via `github_binding.rs`), `AgentKey::issue` /
  `AgentKey::delegate` (attenuation-only sub-key delegation — a child's
  `Caveats` must satisfy `child ⊑ parent` or minting fails with
  `CaveatAmplification`), and `Caveats`: a bounded lattice over six axes
  (`fs_read`, `fs_write`, `exec`, `net`, `max_calls`,
  `valid_for_generation`), each `All | Only(set)`, with `Caveats::top()` as
  the unrestricted identity element.
- `AgentKey::possession_challenge()` + `AgentKey::delegate_external()`:
  lets the mesh certify a public key it does **not** hold the private half
  of (e.g. a phone's Secure Enclave key) — the external party signs a
  fresh challenge to prove possession before a cert is minted. This
  solves the **application-layer** identity/cert problem (proving the
  phone owns a pubkey so the mesh will vouch for it and sign envelopes).
  It does **not** solve the transport-layer problem — see the callout in
  §5.2/§5.5 below; a non-exportable Keystore key cannot, by itself, open
  the QUIC connection that carries the session's traffic.
- `agent-mesh-transport`: authenticated QUIC via `iroh`. The agent's
  ed25519 key doubles as its iroh `EndpointId` via
  `identity::to_iroh_secret(agent) -> SecretKey` (an **owned** secret key
  — `EndpointBuilder::secret_key` takes the raw seed, not an opaque
  signer). `identity.rs` carries this as an explicit open seam:
  `TODO(phone-keystore)` — iroh performs TLS-level handshake signing
  internally, so a non-exportable platform key (Android Keystore / iOS
  Secure Enclave) cannot back an iroh endpoint today. Closing it needs
  either an iroh API accepting a `dyn Signer`, or a phone-side proxy
  endpoint holding an ephemeral, exportable transport key while the
  durable Keystore identity key only ever signs envelopes/certs. This
  proposal takes the latter path — see §5.2. Relay is **explicitly
  disabled** (`endpoint.rs`: "mesh-native discovery is mDNS, not the iroh
  relay") — reachability is LAN-shaped by design.
- `agent-mesh-discovery`: mDNS only (`_agent-mesh._udp.local.`).
- `agent-mesh-bus`: pub/sub + request/reply on top of the transport.
- `agent-mesh-ratchet`: Signal-style Double Ratchet (via `vodozemac`) for
  message-layer forward secrecy on top of the transport's per-connection
  auth. Available, not required for this proposal (see §8).
- Fingerprint = BLAKE3 hash of a 32-byte ed25519 public key. Purely
  cryptographic; the word "fingerprint" in this codebase has never meant
  biometric data.

**agent-bridle**: no separate capability system of its own — it
*consumes* pre-minted `Caveats` (from `$AGENT_BRIDLE_CAVEATS` or
`~/.agent-bridle/config.toml`) and enforces them fail-closed at tool
dispatch via `Gate` (`agent-bridle-core/src/gate.rs`). Missing grant = deny
by default. This Gate pattern — verify a presented capability, fail closed
if absent or insufficient — is the template for the newt-side enforcement
point in §5.4, not a source of new crypto primitives.

**newt-agent**:
- `newt-core/src/agentic/markdown/stream.rs`: `MarkdownStreamWriter<W>`, a
  block-aware streaming writer. It line-buffers incomplete output and
  holds open multi-line constructs (code fences, tables, lists, quotes)
  until they close, then renders the whole block at once. This exists so
  token-by-token LLM output never renders a half-open construct — and it
  is exactly the granularity a remote client wants: whole blocks, not
  bytes.
- TUI design is a deliberate **plain scroller** — no alternate screen, no
  ratatui widget repaint — specifically so behavior is identical over
  SSH, inside tmux, or piped (`docs/decisions/plain_scroller_tui.md`).
  tmux is therefore **not part of the control plane** for this proposal;
  it's simply how Shawn keeps a session alive and reattachable from a
  terminal, orthogonal to whether a phone is also attached via
  agent-mesh.
- `newt-identity`: a thin provisioning wrapper, not a parallel
  implementation — it re-exports `UserKey`/`AgentKey`/`Caveats` directly
  from `agent-mesh-protocol` and adds `load_or_generate`, `session_root`
  (mints a root agent with `Caveats::top()`), and `attenuate`.
- `newt-tui/src/rich_input.rs`: where prompts/choices are presented today.
  Not markdown — a separate interaction surface, which is why it needs
  its own event type (§5.4) rather than riding along in the markdown
  stream.
- `newt-mesh`: an existing, currently-excluded-from-default-workspace
  crate that bridges newt to agent-mesh for peer-inference requests. Not
  the same use case as this proposal (that crate is agent-to-agent
  inference delegation; this proposal is human-to-session attach/control)
  but it establishes that a newt ↔ agent-mesh dependency edge is already
  an accepted shape in this codebase.

## 5. Architecture overview

```
                 ┌─────────────────────────┐
                 │   newt-agent session    │
                 │  (gnuc, tmux pane, ...) │
                 │                         │
  stdout ◄───────┤  MarkdownStreamWriter   │
  (local TTY)    │       │                 │
                 │       ├──► [new] bus sink ──────────┐
                 │       │                             │
                 │  rich_input.rs                       │
                 │       │                             │
                 │       └──► [new] PromptEvent sink ──┤
                 │                                      │
                 │  [new] Session Gate ◄─────────────── │  Reply / Control-claim
                 │   (verifies Caveats on inbound msgs)  │  (agent-mesh-bus, topic
                 └──────────────────────────────────────┘   scoped to this session)
                                    ▲
                                    │  agent-mesh QUIC (LAN mDNS today,
                                    │  optional unicast DNS-SD for VPN)
                                    ▼
                 ┌─────────────────────────┐
                 │   newt-remote (Android) │
                 │  renders Block/Prompt   │
                 │  events natively;       │
                 │  posts Reply /          │
                 │  ControlClaim           │
                 └─────────────────────────┘
```

Four pieces of net-new work, each scoped to a repo:

### 5.1 agent-mesh: assurance-tier caveat axis

Today `Caveats` has no way to express *how* a capability was authenticated
— only *what* it authorizes. Add a seventh axis:

```rust
pub enum Assurance {
    Possession,      // holder signed with the key; nothing more proven
    LocalPin,        // + a local PIN/passphrase gated the key's use
    LocalBiometric,  // + platform biometric (Face/Touch ID, Android
                     // BiometricPrompt, fingerprint) gated the key's use
}
```

A capability mint can *require* a minimum `Assurance` for a given caveat
scope (e.g. "this Control capability is only valid if presented at
`LocalBiometric` or better"). The verifier checks the claimed tier
against what the challenge-response in §4 actually proved.

**Known residual risk, not a solved problem.** The WebAuthn comparison
above is real but incomplete, and the gap matters: WebAuthn's `UV` flag
is backed by an attestation certificate chained to a manufacturer root,
checked at registration and re-affirmed inside signed authenticator data
on every assertion, so a relying party that cares can pin to attested
authenticator classes. `Assurance` here has no equivalent — it's a field
in `AgentMetadata` set by whichever code path calls
`delegate_external(challenge, proof, metadata)`, and `delegate_external`
only verifies the possession proof, never *how* the signing key's use
was gated. A rooted phone or a patched `newt-remote` APK can request
`Assurance::LocalBiometric` without `BiometricPrompt` ever having fired,
and the mesh has no cryptographic way to catch that — it is trusting the
client's platform to "honestly declare" the tier, not proving it the way
WebAuthn's attestation chain does. Accepted as a residual risk for v1
(closing it — e.g. via Android's Play Integrity API or Keystore hardware
attestation to at least bind `LocalBiometric` claims to an attested,
unmodified OS/app — is real work, out of scope here, and should be filed
as an explicit follow-up rather than assumed solved by the analogy).

This is a breaking change to a **published, crates.io-versioned** struct
(0.6.x). See §7 for the migration plan and its own gaps, and note near
the end of §7: this is a separate approval checkpoint from the rest of
this design, not a detail to wave through with everything else.

### 5.2 agent-mesh: two capabilities per session, not one

At session start, `newt-identity`'s `session_root` mints a session-root
`AgentKey` as it does today, then delegates two **sibling** sub-keys from
it via the existing `AgentKey::delegate()` (already supports minting
multiple differently-scoped children from one parent — no new mechanism
needed):

- **Attach** capability: read-only scope on this session's bus topic,
  `Assurance::Possession` sufficient.
- **Control** capability: read-write scope on this session's bus topic,
  requires `Assurance::LocalPin` or `LocalBiometric` (default: biometric;
  see §6 for the pin/no-scanner fallback).

This minting step is mechanical and happens automatically when the
session starts — it requires no human presence check on the host and no
"setup ceremony." The authentication moment that actually matters is
later, when a device *presents* one of these capabilities to join (§5.4)
— that's where `Assurance` gets evaluated, not here.

Each sub-key's fingerprint is handed out separately — "the Attach
fingerprint" and "the Control fingerprint" the way this was originally
described. To be precise about terminology: both are cryptographic key
fingerprints (BLAKE3 hashes of ed25519 public keys), not two biometric
gestures — "fingerprint" never means biometric data in this codebase
(§4). In the actual phone UX this is one `BiometricPrompt` at most: a
device attaches silently at `Possession` tier, and a single biometric
prompt fires only when escalating to Control. Revoking one capability
doesn't touch the other, and both can be regenerated per-session via
`valid_for_generation`.

**Transport binding.** Neither AgentKey binds the QUIC transport
directly — per the `TODO(phone-keystore)` seam in §4, a Keystore-backed
key can't anyway, and binding one of the two sibling keys to the single
QUIC connection would collapse the Attach/Control distinction onto
"whatever tier this device's one connection happens to carry." Instead:
the phone opens **one** QUIC connection
using an ephemeral, ordinary (exportable) transport key generated
per-connection — it authenticates the *device*, not the *capability*.
Attach/Control identity is proven at the application layer, per message,
by signing envelopes with whichever AgentKey (Attach or Control) that
message's authority requires, via `agent_mesh_protocol::MeshSigner` — the
same signer the protocol layer already uses today. The Session Gate (§5.4)
checks the envelope's signer/cert chain, not the transport `EndpointId`.
This is new plumbing (tracked as AM-5 in §9), not something today's
`Endpoint::bind` gives for free.

### 5.3 agent-mesh: unicast DNS-SD as an opt-in discovery path

mDNS discovery doesn't cross WireGuard or Tailscale (neither forwards
multicast by default). Rather than a bespoke "dial a known address"
escape hatch, add unicast DNS-SD (RFC 6763 — the standard wide-area
sibling of mDNS, same record shapes) as a second, explicitly opt-in
discovery backend. Tailscale's MagicDNS can resolve it through the tunnel
without touching multicast at all. This is a setting per the original
ask ("a setting to turn on DNS discovery for remote connections"), off by
default — LAN mDNS remains the default path.

### 5.4 newt-agent: two sinks + a Session Gate

The wire types this section introduces (`Block`, `PromptEvent`, `Reply`,
`ControlClaim`) should be defined **agent-line-agnostic**, not as
"newt's protocol." `gilamonster-agent` already inherits newt-agent's
crates wholesale, and `hermes-thoon` shares the same lineage; if these
types are hardcoded into `newt-core`/`newt-tui` specifically, reusing
this for either later means forking instead of depending on something
shared. Where exactly they should live (a new thin shared crate, or
inside `agent-mesh` itself alongside the bus/protocol types) is an open
question in §11 — but the design intent is that `newt-agent` is the
*first* implementer of this protocol, not its only possible one, and
`newt-remote` the app should only ever need to know the protocol, not
which agent line is on the other end of it.

- Add a second sink to `MarkdownStreamWriter` that publishes each
  completed block to the session's agent-mesh-bus topic, alongside the
  existing stdout write. Additive; the local TTY path is untouched.
- Add a `PromptEvent` type (`ChoicePrompt { options }`,
  `TextInputPrompt { label }`) emitted from `rich_input.rs` alongside
  (not inside) the markdown block stream, with a matching `Reply`
  message type the client posts back.
- Add a Session Gate, modeled on agent-bridle's `Gate` — but modeled on
  the *whole* thing, not just its scope check. The real `Gate`
  (`agent-bridle-core/src/gate.rs`) enforces atomic call budgets,
  generation-scoped revocation, and — the part that matters most here —
  a `consumed: Mutex<HashSet<[u8; 32]>>` single-use ledger that rejects
  replay of an already-accepted discharge. The newt Session Gate needs
  the same discipline for `ControlClaim`/input messages, not just a
  scope + `Assurance` check, or a captured/replayed claim from an earlier
  connection can be re-accepted. Every inbound bus message on the
  session topic must present a capability; the Gate checks scope,
  `Assurance`, and (for anything that mutates control state) the
  consumed-nonce ledger, and fails closed. A `Reply` needs only the
  Attach capability (it's just data flowing back); an actual
  keystroke/input-injection message needs the Control capability at its
  required assurance tier, and only from whichever peer currently holds
  Control (see "explicit take control" below). Sized accordingly in §9
  (NA-3 is an L, not the M it might look like at a glance).
- **Explicit take control**: a `ControlClaim` message, signed by the
  Control capability at its required assurance tier, replaces whoever
  currently holds it — **but only if the claimant's `Assurance` tier is
  at least as strong as the current holder's**. A lower-tier claim (e.g.
  gnuc's tier-3 `Possession`-only fallback from §6) must not silently
  preempt a higher-tier one (e.g. a phone's `LocalBiometric` session) —
  that would be exactly the silent trust downgrade Goal 6 rules out, just
  arriving from the opposite direction. A force-override path can exist
  for the "I really do want to drop back to the weaker device" case, but
  it must be an explicit, visible action, not what `ControlClaim`
  does by default. The session announces any change (to both the local
  TUI and any other attached viewers) rather than silently swapping — no
  merged/concurrent input, per §3.

### 5.5 newt-remote: the Android app

- Renders `Block` and `PromptEvent` messages natively (Compose UI),
  posts `Reply` and `ControlClaim` messages back over the same bus topic.
- The Attach/Control capability private keys live in the Android
  Keystore, hardware-backed (StrongBox where available), the Control key
  gated by `BiometricPrompt`. Neither key ever leaves the Keystore;
  agent-mesh's existing `possession_challenge`/`delegate_external` path
  (§4) is what lets the mesh trust a key it never held, for **cert and
  envelope signing**. It is *not* what opens the QUIC connection — that
  uses the separate ephemeral transport key from AM-5 (§5.2, §9), which
  is ordinary and exportable precisely because it never needs to survive
  past one connection. Getting this split right is real, scoped work
  (NR-1/NR-3 in §9), not a free consequence of `delegate_external`
  already existing.
- **Binding strategy**: bind to `agent-mesh-transport` / `-protocol` /
  `-bus` directly via UniFFI rather than re-implementing the wire
  protocol in Kotlin. `agent-mesh-py` already proves these crates are
  FFI-friendly (via pyo3); UniFFI is the same idea for
  Kotlin/JNI. Reimplementing ed25519/QUIC/envelope-signing in Kotlin
  would be a second place for the crypto to be wrong. This is the
  single biggest reason to keep this in its own repo rather than an
  agent-mesh subdirectory: it's a distinct build target (Gradle +
  Rust-for-Android via `cargo-ndk`), not a Rust workspace member.

## 6. The gnuc problem: no scanner on a headless box

The biometric/PIN check happens on **whichever device holds the private
key material and is asked to use it** — not on the machine being
controlled. A phone's Secure Enclave/Keystore gates *local use* of its own
key; the mesh only ever verifies a signature. So `gnuc`, as the target
being controlled, never needs a scanner — it only verifies incoming
capabilities, exactly as it already verifies cert chains today.

The gap is the other direction: `gnuc` *originating* a control claim (SSH
in directly and want to take control locally, or an autonomous
agent identity on gnuc wants to claim a session). No presence/PIN/
biometric concept exists anywhere in this stack today for that case.
Three tiers, in order of assurance, all expressible as
`Assurance` values from §5.1:

1. **Best available without new hardware assumptions**: a FIDO2 hardware
   key (YubiKey or similar) plugged into gnuc, using **PIN-only** CTAP2
   user verification — this satisfies real UV without needing a
   fingerprint reader. Maps to `Assurance::LocalPin`, hardware-backed.
2. **Middle ground**: a passphrase-gated local signing key (worth
   checking whether gnuc's board has an fTPM to seal it against, rather
   than a bare passphrase-encrypted file). Also `Assurance::LocalPin`,
   software-backed — weaker than (1) (passphrase can be keylogged/shoulder-
   surfed) but far better than an unprotected key on disk.
3. **Honest fallback, and the recommended default for gnuc today**: no
   local UV at all. Mint the Control capability as
   `Assurance::Possession` and let the Gate (§5.4) and both UIs show
   that plainly — a visible "unverified" badge, and a narrower
   `valid_for_generation` window than a biometric-tier grant would get
   — rather than pretending a check happened that didn't. (Note: "shorter
   TTL" here specifically means the `valid_for_generation` mechanism
   `CertChain::verify_at()` actually enforces today — `AgentMetadata`'s
   `expires_at` field is carried in every cert but nothing currently
   reads it against wall-clock time, so it isn't a lever this proposal
   can lean on without separately wiring it up.) This matches the
   instinct that started this discussion ("maybe we just don't do the
   scanner dance... and warn the user"), and it's a reasonable default
   because gnuc is rarely the device a human is physically sitting at in
   this feature's primary use case — it's usually the *target*, not the
   requester.

Recommendation: ship tier 3 as the default, tier 1 as an opt-in upgrade
path if direct-console control from gnuc ever matters enough to justify a
hardware key.

## 7. Trade-offs

**Tap the markdown stream vs. adapt through tmux control mode.** A tmux
`-C` control-mode adapter would need zero newt-agent changes — attach to
the pane, mirror `capture-pane` output, inject via `send-keys`. Rejected
because it puts the Android client back in the terminal-emulation
business (re-deriving block boundaries, styling, and prompt structure
from ANSI text) for no benefit — `MarkdownStreamWriter` already produces
exactly the semantic unit we want. The tap is also a smaller, additive
diff (one new sink) than it might sound. tmux keeps existing purely as
Shawn's local session-persistence tool, decoupled from this feature.

**Client-side vs. server-side (relying-party) biometric verification.**
Server-side would mean gnuc (or any target machine) needs to trust and
interpret biometric material directly — more attack surface, and it
flatly doesn't work on hardware without a scanner. Client-side (the
WebAuthn model) means the target only ever verifies a signature and an
honest self-report of assurance tier from the requester's platform. The
cost is that the mesh has to *trust* a platform's self-reported
`Assurance` value rather than proving it cryptographically end-to-end —
acceptable here because the alternative (remote attestation of biometric
hardware) is disproportionate for this use case, and the existing
possession-challenge mechanism already accepts this class of trust.

**New caveat axis vs. an out-of-band assurance record.** Folding
`Assurance` into `Caveats` keeps a single verification path (one lattice,
one `CertChain::verify()`) instead of two systems that both have to be
checked correctly everywhere. The cost is real: `Caveats` is a public
type in a crate already published to crates.io (0.6.x), and it has a
**second, hand-written mirror** to keep in sync —
`agent-mesh-protocol/src/pyo3_module.rs`'s `PyCaveats` restates the same
axes field-by-field for the Python binding, with its own
`serde_json`-roundtrip contract. Landing the new axis is really two
pieces of work (split as AM-2a/AM-2b in §9, not one AM-2): the Rust-side
lattice change, and the pyo3 binding + its Python-side contract, which
won't update itself.

Landing the Rust side needs either (a) an additive-only change
(`#[serde(default)]`-backed field, plus a minor version bump — pre-1.0
semver allows this) or (b) a genuinely new type wrapping `Caveats` if
additive change turns out to affect the lattice's meet/attenuation laws
in a way that isn't backward compatible. Needs a spike before committing
(issue AM-1 below) — and that spike must also answer the question
`serde(default)` alone doesn't: **what does an absent `Assurance` mean**?
Two skew directions, both real: an *old* cert (minted before this axis
existed) checked by a *new* verifier must not be read as "unattested, so
treat as top/unrestricted" — default it to `Assurance::Possession`, the
most conservative reading, so a pre-existing Control capability doesn't
suddenly satisfy a `LocalBiometric` requirement it never actually proved.
And a *new* cert (carrying a real `Assurance` value) checked by an
*un-upgraded old verifier* (e.g. a pinned `agent-mesh-py` version) will
silently not check `child.assurance ⊑ parent.assurance` at all, because
it doesn't know the field exists — meaning the safety property this axis
adds only holds once every verifier on a given mesh is upgraded. That's
worth stating plainly rather than discovering later.

**This is a separate go/no-go checkpoint.** AM-1/AM-2a/AM-2b constitute a
semver-breaking change to a published, externally-consumed crate — scope
the original request didn't ask for (pubkey encryption + OCAP-style
gating + a fingerprint-shaped check, not a public API break to a shipped
library). Confirm this specifically before AM-1 starts; don't treat it as
bundled into approving the rest of this design.

**Possession-only fallback: honest label vs. silent upgrade.** Already
decided in §6 — label it, don't fake it. Recorded here because it's a
recurring temptation (e.g. "just treat SSH-key possession as good
enough and don't tell anyone") and worth resisting explicitly for every
future device that lacks a scanner, not just gnuc.

**Repo placement: one dedicated repo vs. folding into agent-mesh or
newt-agent.** Chosen: a new repo (`newt-remote`), because the Android app
is a distinct build target and toolchain (Gradle/Kotlin/NDK) that doesn't
belong in either Rust workspace, and because this matches the existing
"many small, focused repos" pattern already used across the
Gilamonster-Foundation org rather than accreting unrelated scope onto
agent-mesh or newt-agent.

## 8. Double Ratchet (agent-mesh-ratchet): not required for v1

`agent-mesh-ratchet` gives message-layer forward secrecy/post-compromise
security on top of transport-layer auth. It's available but not on the
critical path for v1 — with a caveat worth stating precisely rather than
hand-waving: the transport's per-connection QUIC auth gives
confidentiality+integrity for *that connection's* lifetime, not for the
overall session, and the overall session (per §1/§6 — a `tmux` pane on
gnuc, kept alive and reattached to over days) is explicitly **not**
short-lived. What actually stays bounded per-connection is exposure to a
single compromised QUIC session, not exposure across the AgentKey
material's full lifetime — Double Ratchet's forward secrecy and
post-compromise self-healing is precisely for the latter, and a
multi-day attach/detach/reattach pattern is exactly where that matters
more, not less. Accepted as a residual risk for v1 rather than a solved
non-issue: revisit once sessions routinely span multiple reconnects over
more than a day, or sooner if this ships before that's validated.

## 9. Implementation plan (dependency order)

Phased so each step is independently mergeable and testable. "Doc-only"
issues (this file, per-repo READMEs) aren't listed as separate line items.

| # | Repo | Issue | Depends on | Size |
|---|------|-------|------------|------|
| AM-1 | agent-mesh | Spike: additive vs. wrapping-type approach for the `Assurance` axis on `Caveats`; confirm lattice laws hold; **decide absent-`Assurance` semantics both directions** (old cert/new verifier defaults to `Possession`; new cert/old verifier silently skips the check — document it) | — | S |
| AM-2a | agent-mesh | Land the `Assurance` axis in Rust + lattice proptest coverage + version bump | AM-1 | M |
| AM-2b | agent-mesh | Update `PyCaveats` (pyo3) hand-written mirror + its `serde_json` roundtrip contract | AM-2a | S |
| AM-3 | agent-mesh | Session dual-key helper: mint sibling Attach/Control `AgentKey`s from a session root via existing `delegate()` | AM-2a | S |
| AM-4 | agent-mesh | Unicast DNS-SD discovery backend + settings toggle | — (parallel with AM-1..3) | M |
| AM-5 | agent-mesh | Ephemeral transport-key design: phone-side proxy QUIC identity, decoupled from the Attach/Control AgentKeys, closing the `TODO(phone-keystore)` seam in `identity.rs` | AM-3 | M |
| NA-0 | newt-agent | Resolve `agent-mesh` availability in default CI checkouts — `newt-mesh` is currently `exclude`d from the workspace specifically because it isn't (`Cargo.toml`); pick submodule vs. publish-then-path-override vs. lifting the exclusion before any agent-mesh-dependent code lands | — | S |
| NA-1 | newt-agent | Second sink on `MarkdownStreamWriter`: publish completed blocks to the session's bus topic (budget includes an async integration test, not just unit tests — see §10) | NA-0, AM-3 | M |
| NA-2 | newt-agent | `PromptEvent`/`Reply` types wired into `rich_input.rs` | NA-0, AM-3 | S |
| NA-3 | newt-agent | Session Gate, modeled on the **full** agent-bridle `Gate` including its consumed-nonce/replay ledger — not just a scope + `Assurance` check — plus the tier-comparison rule for `ControlClaim` (§5.4) | NA-1, NA-2 | L |
| NA-4 | newt-agent | `ControlClaim` handling: explicit take-control handoff, reject lower-tier preemption of a higher-tier holder, announce to all viewers (integration-test budget) | NA-3 | S |
| NR-1 | newt-remote | Repo/app skeleton: Gradle project, UniFFI bindings for `agent-mesh-{protocol,transport,bus}` via `cargo-ndk` | AM-3, AM-2a (bindings regenerate on axis change) | L |
| NR-2 | newt-remote | Render `Block`/`PromptEvent` natively (Compose); post `Reply` | NR-1, NA-1, NA-2 | M |
| NR-3 | newt-remote | Android Keystore-backed Control key (`BiometricPrompt`-gated) for cert/envelope identity via `possession_challenge`/`delegate_external`; separate ephemeral transport key for QUIC per AM-5 — **not** the same key | NR-1, AM-2a, AM-5 | M |
| NR-4 | newt-remote | `ControlClaim` UI (take control / release) | NR-2, NR-3, NA-4 | S |
| X-1 | newt-agent + newt-remote | End-to-end dogfood: phone attaches to and takes control of a real session on gnuc/beaver over the WireGuard tunnel | all above | S |
| GN-1 (stretch) | agent-mesh or ops | YubiKey PIN-only UV path for gnuc-originated control claims (§6 tier 1) | AM-2a | M |

Suggested execution order: **AM-1 → AM-2a → AM-2b**, with **AM-3** right
behind AM-2a and **AM-4**/**NA-0** any time in parallel (independent
surfaces). **AM-5** follows AM-3. Once AM-3 lands, **NA-1/NA-2** (which
also need NA-0) can run in parallel with each other and with **NR-1**
(the Android skeleton doesn't need the newt-agent side to exist yet, just
the wire types from AM-2a/AM-3, though it will need rebuilding if AM-2a
changes after NR-1 starts). **NA-3** waits on both NA-1 and NA-2, and is
the single biggest newt-agent risk item now that it's sized L. **NR-2**
waits on NR-1 plus its newt-agent counterparts; **NR-3** additionally
waits on AM-5. **X-1** and **GN-1** are last/optional.

## 10. Effort estimate

Sizes above are relative, not calendar time — this is nights/weekends
pace with AI-agent pairing, not a staffed team, so treat these as
session counts (a "session" being a few focused hours):

- **S**: 1–3 sessions
- **M**: 4–8 sessions
- **L**: 10+ sessions, or "expect at least one detour"

By that ruler: the agent-mesh work (AM-1, 2a, 2b, 3, 4, 5) is roughly
3–4 weeks of part-time effort — a week more than an earlier pass at this
estimate, because the review that shook this doc out turned up two items
that were originally invisible: AM-5 (the ephemeral transport-key design,
required because a Keystore-backed key literally cannot bind an iroh
QUIC endpoint today) and the AM-2a/AM-2b split (the pyo3 binding doesn't
update itself). Front-load AM-1 — a wrong call there, or skipping its
absent-`Assurance`-semantics question, costs the most to unwind later.

The newt-agent work (NA-0..4) is similar in overall size but reshaped:
NA-0 is a small but mandatory unblock (agent-mesh isn't reachable from a
stock CI checkout today), and NA-3 is now sized L, not M — a Session
Gate that actually replay-protects `ControlClaim` the way agent-bridle's
real `Gate` does is a genuinely new fail-closed subsystem, not a port.
Both S/M/L sizes above are meant to already include the test-writing time
needed to clear agent-mesh's 75% and newt-agent's 80% workspace coverage
floors (per each repo's `CLAUDE.md`/`AGENTS.md`) — NA-1 and NA-4
specifically will want integration-style async tests, which historically
cost more than the unit tests a bare feature diff implies.

The Android app (NR-1..4) is still the largest unknown by far — NR-1
alone (Rust-for-Android toolchain, UniFFI codegen across a crate whose
public API is still moving during AM-2a, a working Gradle build) is
genuinely new ground compared to everything else in this plan, which is
mostly extending patterns that already exist in these codebases. Budget
NR-1 as the line item most likely to blow its estimate, with NA-3 now a
close second. Total, optimistically: 8–12 weeks part-time to a working
end-to-end dogfood (X-1), realistically closer to 3–4 months once the
Android detour and the newly-surfaced AM-5/NA-3 scope are priced in.

## 11. Open questions for review

- Does the `Assurance` axis belong on `Caveats` at all, or is it cleaner
  as a property of the `AgentKey` delegation itself (i.e. baked into the
  cert, not the lattice)? AM-1 should settle this.
- Should `ControlClaim` require the *current* Control holder to
  acknowledge, or is a signed claim from a valid Control capability
  sufficient to preempt unilaterally? Currently assumed unilateral
  (simpler, matches "explicit take control" as originally described) —
  worth confirming that's actually desired for e.g. Shawn's own two
  devices vs. a scenario with more than one authorized human.
- `newt-mesh` already exists as an excluded-from-default-workspace
  bridge crate for a different purpose (peer inference delegation). Does
  NA-1..4 belong inside `newt-mesh`, or as new modules alongside it? Tied
  to this: where should the agent-line-agnostic `Block`/`PromptEvent`/
  `Reply`/`ControlClaim` protocol types (§5.4) actually live — a new thin
  shared crate, inside `agent-mesh` itself, or inside `newt-mesh` despite
  its narrower current scope? Needs a look at `newt-mesh`'s current shape
  before NA-1/NA-0 start.
