# NIP-XX: NOSTR Mail -- Draft Specification

This directory contains the draft NIP document for NOSTR Mail, an encrypted
asynchronous messaging protocol with economic anti-spam.

## Files

| File | Description |
|------|-------------|
| `nip-xx-nostr-mail.md` | The NIP specification document |

## What This Is

This is a draft NIP (NOSTR Implementation Possibility) for submission to
[nostr-protocol/nips](https://github.com/nostr-protocol/nips). It defines a
protocol for email-like messaging over NOSTR using gift-wrapped encryption
(NIP-44/NIP-59) with Cashu ecash anti-spam.

The protocol introduces six event kinds:

- **Kind 1400** -- Mail message (unsigned rumor, gift-wrapped)
- **Kind 1401** -- Mail receipt (delivery/read confirmation, gift-wrapped)
- **Kind 10050** -- DM relay list (existing, from NIP-17)
- **Kind 10097** -- Spam policy (anti-spam configuration)
- **Kind 30099** -- Mailbox state (read/flag/folder sync)
- **Kind 30016** -- Mail draft (encrypted, addressable)

## How to Review

1. Read the NIP document (`nip-xx-nostr-mail.md`) in full.
2. Compare against the NIP format conventions in existing NIPs (NIP-17, NIP-44,
   NIP-59) for style consistency.
3. Verify that all RFC 2119 conformance language (MUST, SHOULD, MAY) is used
   precisely and consistently.
4. Check that every event kind has at least one JSON example.
5. Verify that the security considerations section addresses the known threat
   model limitations.

## Related Implementations

Two independent implementations exist and have been tested for interoperability:

| Implementation | Language | Location |
|---------------|----------|----------|
| Reference | TypeScript | [nostr-mail-ts](https://github.com/luthen-seas/nostr-mail-ts) |
| Second | Go | [nostr-mail-go](https://github.com/luthen-seas/nostr-mail-go) |

## Test Vectors

Conformance test vectors are published at `test-vectors/`:

| File | Contents |
|------|----------|
| `mail-event.json` | Kind 1400 rumor creation (7 vectors) |
| `gift-wrap.json` | Seal and gift wrap structure |
| `spam-tier.json` | Anti-spam tier evaluation |
| `thread.json` | Threading and tree construction |
| `state.json` | Mailbox state serialization |
| `conformance-spec.md` | Full conformance test specification (47 tests) |

## Specification Highlights

Key design choices incorporated into the NIP:

- Rumor JSON serialization order is not significant (implementations may use any valid JSON)
- Timestamp randomization uses uniform CSPRNG over [-172800, +172800] inclusive range
- Tag positional element rules require empty intermediates (no omission)
- Cashu P2PK pubkey derivation always uses 0x02 prefix (BIP-340 even y)
- Mailbox state tags place event ID at index 1 (NOSTR convention)
- Two-phase Cashu validation: structural (sync) then mint (async)
- Thread tag without reply tag implies direct reply to root
- Deterministic state merge with lexicographic ID tiebreaking
- All tag elements must be JSON strings (numeric values as decimal strings)

## Status

Draft. Not yet submitted to `nostr-protocol/nips`.


---

## Project Layout — NOSTR Mail Ecosystem

The NOSTR Mail project is split across six repositories with clear ownership of each artifact:

| Repo | Source of truth for | This repo? |
|---|---|---|
| [nostr-mail-spec](https://github.com/luthen-seas/nostr-mail-spec) | Living spec, threat model, decisions log, design docs |  |
| [nostr-mail-nip](https://github.com/luthen-seas/nostr-mail-nip) | Submission-ready NIP draft, **canonical test vectors** | ✅ |
| [nostr-mail-ts](https://github.com/luthen-seas/nostr-mail-ts) | TypeScript reference implementation |  |
| [nostr-mail-go](https://github.com/luthen-seas/nostr-mail-go) | Go second implementation (interop) |  |
| [nostr-mail-bridge](https://github.com/luthen-seas/nostr-mail-bridge) | SMTP ↔ NOSTR gateway |  |
| [nostr-mail-client](https://github.com/luthen-seas/nostr-mail-client) | Reference web client (SvelteKit) |  |

**Test vectors** are canonical in `nostr-mail-nip/test-vectors/` and consumed by the implementation repos via git submodule. Do not edit a local copy in an impl repo — submit changes to `nostr-mail-nip`.

See [CONTRIBUTING.md](CONTRIBUTING.md) for the cross-repo contribution workflow, [SECURITY.md](SECURITY.md) for vulnerability reporting, and [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) for community standards.
