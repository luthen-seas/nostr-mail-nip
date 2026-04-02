NIP-XX
======

NOSTR Mail -- Encrypted Asynchronous Messaging with Economic Anti-Spam
----------------------------------------------------------------------

`draft` `optional`

This NIP defines a protocol for encrypted, asynchronous, email-like messaging
over NOSTR with optional economic anti-spam via Cashu ecash micropayments.

## Abstract

This NIP extends the NOSTR gift-wrap messaging model (NIP-17, NIP-44, NIP-59)
to support structured, email-like communication. It introduces kind 1111 mail
messages with subject lines, typed recipients (To/CC/BCC), threaded
conversations, encrypted file attachments via Blossom, and a tiered anti-spam
system that combines social trust signals with proof-of-work and Cashu ecash
micropayments.

The protocol reuses the three-layer encryption model from NIP-59 (rumor, seal,
gift wrap) without modification. All mail content is carried inside unsigned
kind 1111 rumors that are sealed and gift-wrapped before publication. Metadata
visible to relays is limited to the recipient's public key on the outer gift
wrap layer.

NOSTR Mail is designed for asynchronous, store-and-forward messaging between
parties who may not share a social graph. Unlike NIP-17 direct messages, which
target real-time chat between known contacts, NOSTR Mail addresses cold-contact
use cases where the sender may be unknown to the recipient and must demonstrate
legitimacy through economic or computational signals.

## Motivation

NIP-17 private direct messages provide strong privacy guarantees but lack
features required for email-like workflows:

- **No structured metadata.** NIP-17 kind 14 messages have no standard way to
  express subject lines, recipient roles, or content types beyond plain text.
- **No threading model.** Conversations cannot be threaded into hierarchical
  reply chains. NIP-10 `e` tag conventions address short-note replies but do
  not define root-tracking for long-lived threads.
- **No attachment standard.** File sharing via kind 1111 (NIP-17) provides raw
  file metadata but does not define encrypted upload, multi-server redundancy,
  or inline image references.
- **No anti-spam for cold contacts.** When the sender is not in the
  recipient's contact list and has no NIP-05 identity, there is no protocol-
  level mechanism to distinguish legitimate messages from spam. Relays can
  rate-limit, but this is a blunt instrument that penalizes legitimate new
  correspondents equally.
- **No mailbox state sync.** Read/unread status, folder assignments, and flags
  have no standard representation, making multi-device mail clients impossible
  without proprietary sync.
- **No delivery receipts.** Senders have no way to confirm that a message was
  delivered to the recipient's relay or that the recipient has read it.

This NIP addresses all of the above while maintaining full backward
compatibility with the existing NIP-44/NIP-59 encryption infrastructure.

## Dependencies

This NIP depends on the following specifications:

| NIP | Name | Usage |
|-----|------|-------|
| [NIP-01](01.md) | Basic Protocol | Event structure, relay communication |
| [NIP-05](05.md) | DNS Identifiers | Anti-spam tier 1 verification |
| [NIP-09](09.md) | Event Deletion | Requesting deletion of sent mail |
| [NIP-13](13.md) | Proof of Work | Anti-spam tier 2 |
| [NIP-17](17.md) | Private DMs | Base messaging model, kind 10050 relay list |
| [NIP-42](42.md) | Authentication | Relay-enforced inbox access control |
| [NIP-44](44.md) | Encrypted Payloads | Encryption at seal and wrap layers |
| [NIP-59](59.md) | Gift Wrap | Three-layer wrapping protocol |
| [NIP-65](65.md) | Relay List Metadata | Outbox model relay discovery |
| [NIP-94](94.md) | File Metadata | File metadata tag conventions |
| [NIP-B7](b7.md) | Blossom | Decentralized file storage |

Additionally, the anti-spam mechanism references:

| Specification | Usage |
|---------------|-------|
| [Cashu NUT-00](https://github.com/cashubtc/nuts/blob/main/00.md) | Token format (V4) |
| [Cashu NUT-11](https://github.com/cashubtc/nuts/blob/main/11.md) | P2PK spending conditions |

## Specification

### Event Kinds

| Kind | Name | Description | Behavior |
|------|------|-------------|----------|
| 15 | Mail Message | Unsigned rumor containing mail content | Regular (via gift wrap) |
| 1112 | Mail Receipt | Delivery/read confirmation | Regular (via gift wrap) |
| 10050 | DM Relay List | Inbox relay preferences (existing, NIP-17) | Replaceable |
| 10097 | Spam Policy | Anti-spam configuration | Replaceable |
| 10099 | Mailbox State | Read/flag/folder state | Replaceable |
| 30015 | Mail Draft | Encrypted draft messages | Addressable (by `d` tag) |

Kind 1111 and kind 1112 events are never published directly. They exist only as
unsigned rumors inside NIP-59 gift wraps (kind 1059). Kind 10097, 10099, and
30015 events are published as standard signed NOSTR events encrypted to the
author's own public key.

### Mail Message (Kind 1111)

A kind 1111 event is an unsigned rumor (per NIP-59) containing the mail message.
The `sig` field MUST be empty or absent. The `pubkey` field MUST contain the
sender's real public key.

#### Tags

All tag elements MUST be JSON strings. Numeric values (such as file sizes)
MUST be serialized as their decimal string representation.

Tags use positional semantics -- the meaning of each element is determined by
its index within the tag array. When a tag has optional trailing elements:

1. Tags MUST include all positional elements up to and including the last
   non-empty element.
2. Empty intermediate elements MUST be represented as empty strings (`""`),
   not omitted.
3. Trailing empty elements MAY be omitted.

##### Recipient Tags

```
["p", <hex-pubkey>, <relay-hint>, <role>]
```

| Index | Element | Required | Description |
|-------|---------|----------|-------------|
| 0 | `"p"` | MUST | Tag name |
| 1 | pubkey | MUST | 32-byte hex-encoded recipient public key |
| 2 | relay hint | MAY | WebSocket relay URI hint for the recipient |
| 3 | role | SHOULD | One of `"to"`, `"cc"`, or `"bcc"` |

If the role element is absent, implementations MUST treat the recipient as
`"to"`. If the relay hint is not known, it MUST be an empty string `""` when
the role element is present.

Valid examples:

- `["p", "<pubkey>", "wss://relay.example.com", "to"]` -- relay and role
- `["p", "<pubkey>", "", "cc"]` -- no relay hint, CC role
- `["p", "<pubkey>"]` -- no relay, implicit "to" role

Invalid example:

- `["p", "<pubkey>", "cc"]` -- `"cc"` at index 2 is parsed as a relay URI

A mail message MUST contain at least one `p` tag. BCC recipients are included
in the rumor's `p` tags with role `"bcc"`. See [Recipients](#recipients-to-cc-bcc)
for how BCC privacy is maintained at the gift-wrap layer.

##### Subject Tag

```
["subject", <text>]
```

Every kind 1111 rumor MUST include exactly one `subject` tag. The subject text
SHOULD be a short summary of the message (analogous to an email subject line).
Implementations SHOULD limit subject lines to 256 UTF-8 characters.

##### Content-Type Tag

```
["content-type", <mime-type>]
```

Specifies the MIME type of the `content` field. If this tag is absent, the
content type MUST be assumed to be `text/plain`.

Supported content types:

| Content Type | Description |
|-------------|-------------|
| `text/plain` | Default. Plain text, no formatting. |
| `text/markdown` | CommonMark markdown. Recommended for rich formatting. |
| `text/html` | HTML content. Clients MUST sanitize before rendering. |

##### Threading Tags

```
["reply", <event-id>, <relay-hint>]
["thread", <event-id>, <relay-hint>]
```

The `reply` tag identifies the direct parent message being replied to. The
`thread` tag identifies the root message of the conversation thread. Both tags
use the gift-wrap event ID of the referenced message.

Semantics:

- A message with **no** `reply` tag and **no** `thread` tag is a **root
  message** that starts a new conversation.
- A message with **both** `reply` and `thread` tags is a **reply** within an
  existing conversation. The `reply` tag points to the direct parent; the
  `thread` tag points to the thread root.
- A message with a `thread` tag but **no** `reply` tag SHOULD be treated as a
  **direct reply to the root** identified by the `thread` tag. This is a
  shorthand: when the parent is the thread root, the `reply` tag MAY be
  omitted since it would be redundant with the `thread` tag.
- A message with a `reply` tag but **no** `thread` tag SHOULD be treated as a
  reply whose thread root is unknown. Implementations SHOULD attempt to
  discover the thread root by following the reply chain. If the root cannot be
  determined, the message SHOULD be displayed as part of a thread identified
  by the `reply` target.

When replying to a message that is itself the root, both tags reference the
same event ID.

##### Attachment Tags

```
["attachment", <sha256-hash>, <filename>, <mime-type>, <size>]
["attachment-key", <sha256-hash>, <hex-encryption-key>]
["inline", <sha256-hash>, <content-id>]
["blossom", <url1>, <url2>, ...]
```

The `attachment` tag describes a file attachment. The SHA-256 hash is computed
over the encrypted file data (i.e., the Blossom-stored bytes). The `size`
value is the original unencrypted file size in bytes, serialized as a decimal
string.

The `attachment-key` tag provides the AES-256-GCM symmetric key (hex-encoded,
32 bytes) needed to decrypt the file after downloading from Blossom. The key
tag is linked to the attachment by the shared SHA-256 hash at index 1.

The `inline` tag identifies an attachment that is referenced inline within the
message body via a `cid:` URI (e.g., `![alt](cid:chart001)` in markdown). The
content-id at index 2 matches the `cid:` reference in the body. Inline
attachments also require an `attachment-key` tag for decryption.

The `blossom` tag lists one or more Blossom server URLs where attached files
can be retrieved. URLs are deduplicated across all attachments in the message.

##### Anti-Spam Postage Tags

```
["cashu", <serialized-token>]
["cashu-mint", <mint-url>]
["cashu-amount", <amount-string>]
```

See [Anti-Spam](#anti-spam) for the complete specification.

#### Content Field

The `content` field contains the message body. The format is determined by the
`content-type` tag (default: `text/plain`). For `text/html` content, clients
MUST sanitize the HTML before rendering (see [Security Considerations](#security-considerations)).

#### Example: Simple Text Message

```json
{
  "kind": 1111,
  "pubkey": "2c7cc62a697ea3a7826521f3fd34f0cb273693cbe5e9310f35449f43622a6748",
  "created_at": 1711843200,
  "tags": [
    ["p", "98b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a", "", "to"],
    ["subject", "Hello"]
  ],
  "content": "Hi Bob, how are you?",
  "sig": ""
}
```

#### Example: Reply with Threading

```json
{
  "kind": 1111,
  "pubkey": "98b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a",
  "created_at": 1711846800,
  "tags": [
    ["p", "2c7cc62a697ea3a7826521f3fd34f0cb273693cbe5e9310f35449f43622a6748", "", "to"],
    ["subject", "Re: Hello"],
    ["reply", "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa", "wss://relay.alice.com"],
    ["thread", "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa", "wss://relay.alice.com"]
  ],
  "content": "Hi Alice, I'm doing well! Thanks for asking.",
  "sig": ""
}
```

#### Example: Message with Attachment

```json
{
  "kind": 1111,
  "pubkey": "2c7cc62a697ea3a7826521f3fd34f0cb273693cbe5e9310f35449f43622a6748",
  "created_at": 1711843200,
  "tags": [
    ["p", "98b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a", "", "to"],
    ["subject", "Report attached"],
    ["attachment", "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855", "Q3-Report.pdf", "application/pdf", "2048576"],
    ["attachment-key", "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855", "abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789"],
    ["blossom", "https://blossom.example.com"]
  ],
  "content": "Hi Bob, please find the Q3 report attached.",
  "sig": ""
}
```

#### Example: Message with Cashu Anti-Spam Postage

```json
{
  "kind": 1111,
  "pubkey": "2c7cc62a697ea3a7826521f3fd34f0cb273693cbe5e9310f35449f43622a6748",
  "created_at": 1711843200,
  "tags": [
    ["p", "98b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a", "", "to"],
    ["subject", "Introduction"],
    ["cashu", "cashuBo2FteCJodHRwczovL21pbnQuZXhhbXBsZS5jb20iYXVjc2F0YXCBomFp..."],
    ["cashu-mint", "https://mint.example.com"],
    ["cashu-amount", "21"]
  ],
  "content": "Hi Bob, we met at the conference. Here's my contact info.",
  "sig": ""
}
```

### Encryption

NOSTR Mail uses the three-layer NIP-59 gift wrap model without modification.
The layers are:

1. **Rumor (kind 1111)** -- The unsigned mail message. Contains the sender's
   real public key, all tags, and the message body. Not signed.
2. **Seal (kind 13)** -- The rumor serialized to JSON and encrypted with
   NIP-44 using ECDH between the sender's private key and the recipient's
   public key. Signed by the sender. Tags MUST be empty.
3. **Gift Wrap (kind 1059)** -- The seal serialized to JSON and encrypted
   with NIP-44 using ECDH between a fresh ephemeral private key and the
   recipient's public key. Signed by the ephemeral key. Contains a single
   `["p", <recipient-pubkey>]` tag for routing.

#### Rumor Serialization

When serializing a kind 1111 rumor to JSON for NIP-44 encryption,
implementations MAY use any valid JSON serialization. JSON object key order is
NOT significant. The recipient MUST parse the decrypted JSON string and
extract fields by key name, not by position. Two implementations encrypting
the same rumor will produce different ciphertext (due to both serialization
differences and NIP-44's random nonce), but the decrypted and parsed rumor
MUST be semantically identical.

#### Timestamp Randomization

The `created_at` field on seal (kind 13) and gift wrap (kind 1059) events MUST
be randomized to prevent timing correlation. The randomized timestamp MUST be
computed as:

```
randomized_created_at = actual_unix_timestamp + random_offset
```

where `random_offset` is a uniform random integer in the inclusive range
`[-172800, +172800]` (plus or minus 2 days in seconds). Implementations MUST
use a cryptographically secure random number generator (CSPRNG). The
distribution SHOULD be uniform over the full range.

The rumor's `created_at` field SHOULD reflect the actual time for proper
message ordering within a conversation.

#### Ephemeral Key Requirements

Implementations MUST generate a new random keypair for each gift wrap event.
Ephemeral private keys MUST be discarded after signing the gift wrap.
Implementations MUST NOT reuse ephemeral keys across different gift wraps,
even when wrapping the same rumor for multiple recipients.

### Recipients (To, CC, BCC)

Recipient roles are encoded in the `p` tags of the kind 1111 rumor.

#### To and CC

Recipients with roles `"to"` and `"cc"` are visible to all recipients of the
message. After decryption, every recipient sees the same set of `p` tags in
the rumor.

#### BCC

BCC recipients are included in the rumor's `p` tags with role `"bcc"`. This
means the BCC recipient can see who else was BCC'd, but the mechanism for
hiding BCC recipients from To/CC recipients is at the gift-wrap layer:

To implement BCC:

1. Create a **visible rumor** containing only the `"to"` and `"cc"` recipient
   `p` tags. Seal and gift-wrap this rumor for each To and CC recipient, and
   for the sender's self-copy.
2. Create a **BCC rumor** containing all `p` tags (To, CC, and BCC). Seal and
   gift-wrap this rumor for each BCC recipient individually.

This ensures that To/CC recipients cannot see BCC recipients (they receive a
rumor without BCC `p` tags), while each BCC recipient receives a rumor that
includes the full recipient list.

#### Self-Copy

The sender MUST create a separate gift wrap addressed to themselves (with
their own public key in the outer `p` tag) so that sent messages can be
recovered from relays. The self-copy uses a unique ephemeral key.

### Threading

Thread reconstruction uses the `reply` and `thread` tags to build a
hierarchical conversation tree.

#### Thread Reconstruction Algorithm

Given a set of decrypted mail messages:

1. Create a lookup table mapping each message's gift-wrap event ID to the
   message.
2. For each message with a `replyTo` reference:
   a. If the referenced parent exists in the lookup table, link the message
      as a child of that parent.
   b. If the referenced parent does not exist, treat the message as a root.
3. Messages with no `reply` or `thread` tags are roots.
4. Messages with a `thread` tag but no `reply` tag are children of the thread
   root (if it exists in the set), otherwise roots.
5. Sort children at each level by `created_at` ascending.

#### Thread Grouping

Messages are grouped into conversations by their `thread` tag value. Messages
without a `thread` tag that are not referenced as a parent by any other message
form single-message threads keyed by their own event ID.

### Attachments

File attachments are stored externally on Blossom servers and referenced by
SHA-256 hash in the rumor's tags.

#### Encryption

Attachments MUST be encrypted before upload. The encryption scheme is
AES-256-GCM:

1. Generate a random 32-byte symmetric key using a CSPRNG.
2. Generate a random 12-byte initialization vector (IV).
3. Encrypt the file data with AES-256-GCM using the key and IV.
4. Prepend the IV to the ciphertext: `IV (12 bytes) || ciphertext || GCM auth tag (16 bytes)`.
5. Compute the SHA-256 hash of the encrypted output (this is the Blossom
   storage hash).
6. Upload the encrypted data to one or more Blossom servers.
7. Include `attachment`, `attachment-key`, and `blossom` tags in the rumor.

The `attachment-key` tag carries the hex-encoded AES-256-GCM key. The
encryption key is transmitted inside the encrypted rumor and is therefore
protected by the NIP-44/NIP-59 envelope.

#### Download and Verification

To retrieve an attachment:

1. Download the encrypted data from a Blossom server by hash.
2. Verify the SHA-256 hash of the downloaded data matches the `attachment` tag
   hash. If it does not match, discard the data and try another Blossom server.
3. Decrypt using the key from the `attachment-key` tag.

If all Blossom servers are unreachable, the client SHOULD display the message
with the attachment shown as unavailable and offer a retry option.

### Anti-Spam

The anti-spam system uses a tiered trust model. When a gift-wrapped mail
message is received and decrypted, the client evaluates the sender against
the following tiers in order. The first tier that matches determines the
message's disposition.

#### Tier Model

| Tier | Signal | Disposition | Description |
|------|--------|-------------|-------------|
| 0 | Contact | Inbox | Sender is in recipient's kind 3 follow list |
| 1 | NIP-05 | Inbox | Sender has a verified NIP-05 identifier |
| 2 | PoW | Inbox | Gift wrap has NIP-13 PoW >= policy threshold |
| 3 | Cashu | Inbox | Message includes valid Cashu P2PK postage |
| 4 | (Reserved) | -- | Reserved for future trust signals |
| 5 | Unknown | Quarantine/Reject | No qualifying signal |

Tier evaluation is ordered by priority: a sender who qualifies for tier 0
(contact) is classified as tier 0 even if they also attached a Cashu token.
This prevents unnecessary token redemption.

#### Cashu Postage Token

When none of the free tiers (0, 1, 2) qualify, the sender MAY attach a Cashu
ecash token as "postage" to gain inbox access.

The token MUST be serialized in NUT-00 V4 format in the `cashu` tag. The
`cashu-mint` and `cashu-amount` tags provide metadata for quick evaluation
without parsing the token.

##### P2PK Spending Condition

Anti-spam postage tokens MUST be locked via NUT-11 P2PK (Pay-to-Public-Key)
to the recipient's public key. Bearer tokens (without P2PK) MUST be rejected.

The P2PK spending condition uses the compressed SEC1 encoding of the
recipient's secp256k1 public key. To derive the compressed SEC1 pubkey from a
NOSTR x-only pubkey (BIP-340):

```
compressed_pubkey = 0x02 || nostr_x_only_pubkey
```

The `0x02` prefix (even y-coordinate) is always correct for NOSTR public keys
because BIP-340 specifies that x-only keys implicitly represent the point with
even y-coordinate. Implementations MUST NOT use the `0x03` prefix.
Implementations MUST NOT attempt to compute the actual y-coordinate parity.

**Example:**
- NOSTR x-only pubkey: `98b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a`
- Compressed SEC1 pubkey for P2PK: `0298b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a`

##### Two-Phase Validation

Cashu token validation occurs in two phases:

**Phase 1 -- Structural Validation (synchronous):**

Performed during tier evaluation. Implementations MUST check:

1. The `cashu` tag contains a parseable NUT-00 V4 token.
2. The token amount meets the policy minimum (`cashu-min-sats`).
3. The token's mint URL is in the accepted mints list (if the policy specifies
   one).
4. The token includes a P2PK spending condition (NUT-11).
5. The P2PK lock target matches the recipient's compressed SEC1 pubkey.

If structural validation passes, the message is tentatively classified as
tier 3 and delivered to the inbox.

**Phase 2 -- Mint Validation (asynchronous):**

Performed after inbox delivery. The client SHOULD attempt to swap the token at
the mint (`POST /v1/swap`) to verify the token is unspent and claim the funds.
If the swap fails:

- The message SHOULD be reclassified to tier 5 (quarantine).
- The client SHOULD display a warning that the postage token was invalid.
- The client MUST NOT automatically delete the message.

**Rationale for two phases:** Contacting the mint requires network access and
may be slow or fail. Tier evaluation SHOULD be fast and deterministic. Deferring
mint validation ensures messages are not lost due to temporary mint downtime.

### Spam Policy (Kind 10097)

A kind 10097 replaceable event published by a user declares their anti-spam
preferences. Clients sending mail to this user SHOULD respect these
preferences.

#### Tags

| Tag | Format | Description |
|-----|--------|-------------|
| `contacts-free` | `["contacts-free", "true"\|"false"]` | Contacts bypass all checks (default: `"true"`) |
| `nip05-free` | `["nip05-free", "true"\|"false"]` | NIP-05 senders bypass PoW/Cashu (default: `"true"`) |
| `pow-min-bits` | `["pow-min-bits", "<n>"]` | Minimum NIP-13 difficulty bits (default: `"20"`) |
| `cashu-min-sats` | `["cashu-min-sats", "<n>"]` | Minimum Cashu postage in sats (default: `"1"`) |
| `accepted-mint` | `["accepted-mint", "<url>"]` | Accepted Cashu mint URL (repeatable) |
| `unknown-action` | `["unknown-action", "quarantine"\|"reject"]` | Action for tier 5 (default: `"quarantine"`) |

If no kind 10097 event is published, clients SHOULD use the following defaults:

- `contacts-free`: `true`
- `nip05-free`: `true`
- `pow-min-bits`: `20`
- `cashu-min-sats`: `1`
- `accepted-mint`: (empty -- accept all mints)
- `unknown-action`: `quarantine`

#### Example: Spam Policy Event

```json
{
  "id": "<event-id>",
  "pubkey": "<recipient-pubkey>",
  "created_at": 1711843200,
  "kind": 10097,
  "tags": [
    ["contacts-free", "true"],
    ["nip05-free", "true"],
    ["pow-min-bits", "20"],
    ["cashu-min-sats", "10"],
    ["accepted-mint", "https://mint.minibits.cash"],
    ["accepted-mint", "https://mint.coinos.io"],
    ["unknown-action", "quarantine"]
  ],
  "content": "",
  "sig": "<signature>"
}
```

### Mailbox State (Kind 10099)

A kind 10099 replaceable event stores the user's mailbox state: which messages
have been read, flagged, moved to folders, or deleted. This event is encrypted
to the user's own public key using NIP-44 and published to the user's relays.

#### Tags

| Tag | Format | Description |
|-----|--------|-------------|
| `read` | `["read", "<event-id>"]` | Message marked as read |
| `flag` | `["flag", "<event-id>", "<flag1>", "<flag2>", ...]` | Flags on a message |
| `folder` | `["folder", "<event-id>", "<folder-name>"]` | Folder assignment |
| `deleted` | `["deleted", "<event-id>"]` | Message marked as deleted |

The event ID used in all state tags is the gift-wrap event ID (the kind 1059
event's `id` field), which is the stable identifier the client uses to track
messages.

The `folder` tag places the event ID at index 1 and the folder name at index 2.
This follows NOSTR convention where the primary indexed value comes first after
the tag name.

Standard folder names: `"inbox"`, `"sent"`, `"drafts"`, `"archive"`, `"trash"`.
Clients MAY support custom folder names.

#### Read State (G-Set)

The `read` tags form a Grow-only Set (G-Set) -- a CRDT that supports only
additions, never removals. Once a message ID appears in a `read` tag, it MUST
NOT be removed in any subsequent version of the kind 10099 event. This ensures
convergence across devices: any device marking a message as read propagates
to all other devices on merge.

Implementations MUST NOT provide an "mark as unread" operation that removes
from the read set. If an "unread" UI state is desired, use a separate
mechanism (e.g., a `"needs-attention"` flag).

#### Conflict Resolution

Kind 10099 is a replaceable event. When a client receives multiple versions,
it MUST keep the one with the latest `created_at`. If two state events have
the same `created_at`, the event with the lexicographically lower `id`
(lowercase hex comparison) MUST be kept.

#### Multi-Device Merge

When a client has local state changes that have not been published and
receives a remote state event newer than the last-published local state, the
client SHOULD merge using the following rules:

- **`read`**: G-Set union (add all read markers from both local and remote).
- **`deleted`**: G-Set union (add all deletion markers from both).
- **`flag`**: Union of flag sets per event ID (keep all flags from both).
- **`folder`**: Last-write-wins. The remote state's folder assignments take
  precedence for any event ID present in both local and remote. Local-only
  folder assignments are preserved.

After merging, the client SHOULD publish a new kind 10099 event containing the
merged state.

#### Example: Mailbox State Event

```json
{
  "id": "<event-id>",
  "pubkey": "<user-pubkey>",
  "created_at": 1711900000,
  "kind": 10099,
  "tags": [
    ["read", "aabbccdd11223344aabbccdd11223344aabbccdd11223344aabbccdd11223344"],
    ["read", "eeff00112233445566778899aabbccddeeff00112233445566778899aabbccdd"],
    ["flag", "aabbccdd11223344aabbccdd11223344aabbccdd11223344aabbccdd11223344", "starred"],
    ["folder", "eeff00112233445566778899aabbccddeeff00112233445566778899aabbccdd", "archive"],
    ["deleted", "00112233445566778899aabbccddeeff00112233445566778899aabbccddeeff"]
  ],
  "content": "",
  "sig": "<signature>"
}
```

### Mail Receipt (Kind 1112)

A kind 16 event is an unsigned rumor (delivered via gift wrap) that confirms
message delivery or read status.

#### Tags

| Tag | Format | Description |
|-----|--------|-------------|
| `p` | `["p", "<pubkey>"]` | Original sender's pubkey (receipt recipient) |
| `e` | `["e", "<event-id>"]` | Gift-wrap event ID of the acknowledged message |
| `status` | `["status", "<type>"]` | One of `"delivered"` or `"read"` |

A `"delivered"` receipt indicates the message was received and decrypted. A
`"read"` receipt indicates the user has viewed the message.

Clients MAY send receipts. Clients MUST NOT require receipts for correct
operation.

#### Example: Read Receipt

```json
{
  "kind": 1112,
  "pubkey": "98b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a",
  "created_at": 1711850000,
  "tags": [
    ["p", "2c7cc62a697ea3a7826521f3fd34f0cb273693cbe5e9310f35449f43622a6748"],
    ["e", "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"],
    ["status", "read"]
  ],
  "content": "",
  "sig": ""
}
```

This rumor is sealed and gift-wrapped to the original sender (Alice), who
can then display a "read" indicator.

### Drafts (Kind 30015)

A kind 30015 addressable event stores an encrypted draft message. The `d` tag
provides a stable identifier for updating the draft.

The `content` field contains the draft rumor JSON encrypted with NIP-44 to the
author's own public key. Tags on the outer event are minimal:

| Tag | Format | Description |
|-----|--------|-------------|
| `d` | `["d", "<draft-id>"]` | Unique identifier for this draft |
| `subject` | `["subject", "<text>"]` | Draft subject (for listing without decryption) |

The `subject` tag on the outer event is OPTIONAL and is provided as a
convenience for listing drafts without decryption. It leaks metadata and
SHOULD be omitted if privacy is a concern.

#### Example: Draft Event

```json
{
  "id": "<event-id>",
  "pubkey": "<user-pubkey>",
  "created_at": 1711843200,
  "kind": 30015,
  "tags": [
    ["d", "draft-001"],
    ["subject", "Q3 Report"]
  ],
  "content": "<NIP-44 encrypted draft rumor JSON>",
  "sig": "<signature>"
}
```

### Protocol Flow

#### Sending a Mail Message

```
Alice wants to send a mail message to Bob (to) and Charlie (cc):

Step 1: Create the kind 1111 rumor (unsigned)
  - Set pubkey to Alice's real pubkey
  - Add p tags: ["p", bob, "", "to"], ["p", charlie, "", "cc"]
  - Add ["subject", "Meeting Notes"]
  - Set content to the message body
  - Set created_at to the current time

Step 2: Look up recipient relay preferences
  - Query each recipient's kind 10050 event for inbox relays
  - Fall back to kind 10002 (NIP-65) if no kind 10050 exists

Step 3: Look up recipient's spam policy (optional)
  - Query Bob's kind 10097 event
  - If PoW or Cashu is required and Alice is not in Bob's contacts,
    attach the appropriate anti-spam signal

Step 4: For each recipient (Bob, Charlie) AND for Alice (self-copy):
  a. Serialize the rumor to JSON
  b. NIP-44 encrypt the JSON with ECDH(alice_privkey, recipient_pubkey)
  c. Create a kind 13 seal:
     - pubkey = Alice's pubkey
     - content = encrypted rumor
     - tags = []
     - created_at = now + random_offset (CSPRNG, [-172800, +172800])
     - Sign with Alice's key
  d. Generate a fresh ephemeral keypair
  e. NIP-44 encrypt the seal JSON with ECDH(ephemeral_privkey, recipient_pubkey)
  f. Create a kind 1059 gift wrap:
     - pubkey = ephemeral pubkey
     - content = encrypted seal
     - tags = [["p", recipient_pubkey]]
     - created_at = now + random_offset (CSPRNG, [-172800, +172800])
     - Sign with ephemeral key
  g. Discard the ephemeral private key

Step 5: Publish each gift wrap to the respective recipient's inbox relays
```

#### Receiving a Mail Message

```
Bob receives a kind 1059 event from a relay:

Step 1: Decrypt the gift wrap
  - conversation_key = ECDH(bob_privkey, giftwrap.pubkey)
  - seal_json = NIP-44_decrypt(conversation_key, giftwrap.content)

Step 2: Verify and decrypt the seal
  - Parse the kind 13 seal event
  - Verify the seal's Schnorr signature
  - conversation_key = ECDH(bob_privkey, seal.pubkey)
  - rumor_json = NIP-44_decrypt(conversation_key, seal.content)

Step 3: Validate the rumor
  - Parse the kind 1111 rumor
  - Verify rumor.pubkey == seal.pubkey (sender consistency)
  - Verify rumor.kind == 1111

Step 4: Evaluate anti-spam tier
  - Check if seal.pubkey is in Bob's kind 3 follow list (tier 0)
  - Check if seal.pubkey has a valid NIP-05 (tier 1)
  - Check gift wrap PoW bits against policy (tier 2)
  - Check cashu tag for valid P2PK token (tier 3, structural only)
  - If no tier qualifies, apply the unknown-action policy (tier 5)

Step 5: Process the message
  - Parse tags: subject, recipients, threading, attachments
  - Store in the appropriate folder based on tier evaluation
  - Update kind 10099 state if needed
  - Optionally send a kind 16 delivery receipt
```

### Relay Behavior

Relays that support NOSTR Mail SHOULD implement the following:

- Relays MUST store kind 1059 events per standard NIP-01 behavior.
- Relays SHOULD restrict access to kind 1059 events by requiring NIP-42
  AUTH and only serving them to the pubkey in the `p` tag. This prevents
  unauthorized parties from enumerating a user's encrypted mail.
- Relays MAY rate-limit kind 1059 events per recipient pubkey. When
  rate-limiting, relays SHOULD apply limits per sender-recipient pair (using
  the gift wrap's `p` tag as recipient) to prevent a single spammer from
  exhausting another user's rate limit quota.
- Relays MAY require NIP-13 proof-of-work on kind 1059 events.
- Relays MUST NOT inspect, decrypt, or log the content of kind 1059 events.
- Relays MUST store kind 10097, 10099, and 30015 events per their respective
  replaceable/addressable event semantics.
- Relays MAY enforce a maximum event size for kind 1059 events. The
  recommended maximum is 64 KB.

### Client Behavior

Clients implementing NOSTR Mail:

- Clients MUST use NIP-44 version 2 for all encryption operations.
- Clients MUST use NIP-59 gift wrapping for all kind 1111 and kind 1112 events.
- Clients MUST generate a new ephemeral keypair for each gift wrap.
- Clients MUST randomize timestamps on seal and gift wrap layers using a
  CSPRNG with uniform distribution over [-172800, +172800].
- Clients MUST create a self-addressed gift wrap copy of every sent message.
- Clients MUST verify that rumor.pubkey matches seal.pubkey when unwrapping.
- Clients MUST include at least one `p` tag and one `subject` tag in every
  kind 1111 rumor.
- Clients MUST look up recipient inbox relays via kind 10050, falling back to
  kind 10002 (NIP-65).
- Clients SHOULD publish a kind 10050 event listing preferred inbox relays.
- Clients SHOULD publish a kind 10097 event declaring anti-spam preferences.
- Clients SHOULD evaluate anti-spam tiers in priority order (0 through 5).
- Clients SHOULD support markdown content type (`text/markdown`).
- Clients SHOULD support encrypted file attachments via Blossom.
- Clients SHOULD deduplicate received gift wraps by event ID.
- Clients SHOULD support thread grouping and hierarchical display.
- Clients MAY send kind 16 delivery and read receipts.
- Clients MAY support kind 30015 drafts.
- Clients MUST sanitize HTML content before rendering. At minimum, clients
  MUST strip `<script>`, `<iframe>`, `<object>`, `<embed>`, `<form>`,
  `<input>`, `<style>`, and all `on*` event handler attributes. Clients
  SHOULD use an allowlist-based sanitizer rather than a blocklist.
- Clients MUST NOT sign the kind 1111 or kind 16 rumor events.

### SMTP Bridge (Informational)

_This section is informational and not normative._

An SMTP bridge allows NOSTR Mail to interoperate with traditional email. The
bridge operates as both an SMTP server (receiving email) and an SMTP client
(sending email), translating between the two formats.

#### Inbound (SMTP to NOSTR)

The bridge receives email via SMTP, maps the sender to a NOSTR pubkey (via
NIP-05 reverse lookup or a local mapping table), constructs a kind 1111 rumor
with appropriate tags, and gift-wraps it for the NOSTR recipient.

#### Outbound (NOSTR to SMTP)

The bridge monitors the user's inbox relays for kind 1059 events, unwraps
them, and forwards the content as a standard MIME email to the destination
SMTP server. Attachments are downloaded from Blossom and included as MIME
attachments.

#### Considerations

- The bridge is a trusted intermediary and has access to plaintext content.
- End-to-end encryption is broken at the bridge boundary.
- The bridge SHOULD sign inbound NOSTR messages with its own keypair and
  include a `["bridge", "<bridge-nip05>"]` tag for transparency.

## Test Vectors

Test vectors are published at:

- `impl/test-vectors/mail-event.json` -- Kind 1111 rumor creation vectors
- `impl/test-vectors/gift-wrap.json` -- Seal and gift wrap structure vectors
- `impl/test-vectors/spam-tier.json` -- Anti-spam tier evaluation vectors
- `impl/test-vectors/thread.json` -- Threading and tree construction vectors
- `impl/test-vectors/state.json` -- Mailbox state serialization vectors

### Test Keys

The following keys are used across all test vectors:

| Identity | Private Key (hex) | Public Key (hex) |
|----------|-------------------|------------------|
| Alice | `7f7ff03d123792d6ac594bfa67bf6d0c0ab55b6b1fdb6249303fe861f1ccba9a` | `2c7cc62a697ea3a7826521f3fd34f0cb273693cbe5e9310f35449f43622a6748` |
| Bob | `c15d2a640a7bd00f291e074e5e40419e08593833a5b9bd1b4e89100ef750fa35` | `98b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a` |
| Charlie | `a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2` | `d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e4` |

### Inline Example: Complete Gift Wrap Structure

Alice sends "Hi Bob, how are you?" to Bob:

**Layer 1 -- Rumor (kind 1111, unsigned):**
```json
{
  "kind": 1111,
  "pubkey": "2c7cc62a697ea3a7826521f3fd34f0cb273693cbe5e9310f35449f43622a6748",
  "created_at": 1711843200,
  "tags": [
    ["p", "98b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a", "", "to"],
    ["subject", "Hello"]
  ],
  "content": "Hi Bob, how are you?",
  "sig": ""
}
```

**Layer 2 -- Seal (kind 13, signed by Alice):**
```json
{
  "id": "<sha256-hash>",
  "pubkey": "2c7cc62a697ea3a7826521f3fd34f0cb273693cbe5e9310f35449f43622a6748",
  "created_at": 1711756800,
  "kind": 13,
  "tags": [],
  "content": "<NIP-44 encrypted rumor JSON>",
  "sig": "<schnorr-signature>"
}
```
Note: `created_at` is randomized (differs from rumor). Tags are always empty.

**Layer 3 -- Gift Wrap (kind 1059, signed by ephemeral key):**
```json
{
  "id": "<sha256-hash>",
  "pubkey": "<ephemeral-pubkey>",
  "created_at": 1711670400,
  "kind": 1059,
  "tags": [
    ["p", "98b30d5bfd1e2e751d7a57e7a58e67e15b3f2e0a90f9f7e8e40f7f6e5d4c3b2a"]
  ],
  "content": "<NIP-44 encrypted seal JSON>",
  "sig": "<ephemeral-signature>"
}
```
Note: `pubkey` is a one-time ephemeral key. `created_at` is independently
randomized. Only the `p` tag reveals this is destined for Bob.

## Security Considerations

### Threat Model

NOSTR Mail provides confidentiality and sender authentication against passive
network observers and honest-but-curious relays. It does NOT protect against:

- **Key compromise:** If a user's private key is compromised, all past and
  future messages encrypted to that key can be decrypted. There is no forward
  secrecy.
- **Recipient metadata:** The recipient's pubkey is visible in the gift wrap's
  `p` tag. An observer can determine _who_ is receiving messages (but not from
  whom or what the content is).
- **Traffic analysis:** Message timing, size, and frequency patterns may reveal
  information about communication relationships, even with timestamp
  randomization.
- **Active relay attacks:** A malicious relay could drop, delay, or duplicate
  messages. It cannot forge or modify encrypted content.

### No Forward Secrecy

NIP-44 uses static-key ECDH. The conversation key between any two parties is
deterministic. If either party's private key is later compromised, all past
messages between them can be decrypted. Users requiring forward secrecy should
use an out-of-band key exchange or MLS-based messaging (NIP-EE).

### CSPRNG Requirement

All random values in this protocol -- ephemeral keys, timestamp offsets,
AES-256-GCM keys, AES-256-GCM IVs, and NIP-44 nonces -- MUST be generated
using a cryptographically secure pseudorandom number generator. Use of
`Math.random()` or equivalent non-cryptographic sources is a critical security
vulnerability.

### HTML Sanitization

When `content-type` is `text/html`, the content MUST be sanitized before
rendering. Rendering unsanitized HTML enables cross-site scripting (XSS),
phishing overlays, tracking pixels, and other attacks. Clients MUST strip:

- All `<script>` elements
- All `<iframe>`, `<object>`, `<embed>`, `<applet>` elements
- All `<form>`, `<input>`, `<button>`, `<select>`, `<textarea>` elements
- All `<style>` elements and `style` attributes (or sanitize CSS separately)
- All `on*` event handler attributes (`onclick`, `onload`, `onerror`, etc.)
- All `javascript:` URIs
- All `data:` URIs (except for small inline images, which SHOULD be allowlisted
  by MIME type)
- All external resource references (`<img src="https://...">`) unless
  explicitly allowed by the user, as they leak IP addresses

Clients SHOULD use an allowlist-based sanitizer (e.g., DOMPurify) rather than
a blocklist.

### Cashu P2PK Rationale

Bearer tokens (without P2PK) are vulnerable to front-running: a malicious
relay or network observer could extract the token and redeem it before the
intended recipient. P2PK ensures only the holder of the recipient's private
key can spend the token.

### Attachment Encryption

Files MUST be encrypted before upload to Blossom servers. Uploading plaintext
files to a public Blossom server exposes the content to the server operator
and anyone with the hash. The AES-256-GCM key is carried inside the encrypted
rumor, so it is protected by the NIP-44/NIP-59 envelope.

### Event Size

Gift-wrapped mail events with many attachment tags, long Cashu tokens, and
multiple Blossom URLs can approach relay size limits. Implementations SHOULD
target a maximum kind 1059 event size of 64 KB. If the rumor exceeds this
limit, implementations SHOULD warn the user.

## Backward Compatibility

- This NIP does NOT affect existing NIP-17 direct messages. Kind 14 (DM) and
  kind 1111 (mail) are distinct event kinds with different semantics.
- Kind 1111 and kind 1112 events are always gift-wrapped as kind 1059 events.
  Existing relays that support kind 1059 (for NIP-17 DMs) will store and
  serve NOSTR Mail events without modification.
- Clients that do not implement this NIP will see kind 1059 events but, upon
  decrypting the inner rumor, will encounter kind 1111 and can safely ignore it
  as an unrecognized kind.
- Kind 10097 (spam policy), 10099 (mailbox state), and 30015 (drafts) are new
  event kinds that do not conflict with any existing kinds.

## References

- [NIP-01: Basic Protocol](https://github.com/nostr-protocol/nips/blob/master/01.md)
- [NIP-05: Mapping Nostr keys to DNS-based internet identifiers](https://github.com/nostr-protocol/nips/blob/master/05.md)
- [NIP-09: Event Deletion Request](https://github.com/nostr-protocol/nips/blob/master/09.md)
- [NIP-13: Proof of Work](https://github.com/nostr-protocol/nips/blob/master/13.md)
- [NIP-17: Private Direct Messages](https://github.com/nostr-protocol/nips/blob/master/17.md)
- [NIP-42: Authentication of clients to relays](https://github.com/nostr-protocol/nips/blob/master/42.md)
- [NIP-44: Versioned Encryption](https://github.com/nostr-protocol/nips/blob/master/44.md)
- [NIP-59: Gift Wrap](https://github.com/nostr-protocol/nips/blob/master/59.md)
- [NIP-65: Relay List Metadata](https://github.com/nostr-protocol/nips/blob/master/65.md)
- [NIP-94: File Metadata](https://github.com/nostr-protocol/nips/blob/master/94.md)
- [Blossom (BUD-01 through BUD-06)](https://github.com/hzrd149/blossom)
- [Cashu NUT-00: Token Format](https://github.com/cashubtc/nuts/blob/main/00.md)
- [Cashu NUT-11: Pay-to-Public-Key](https://github.com/cashubtc/nuts/blob/main/11.md)
- [RFC 2119: Key words for use in RFCs](https://www.rfc-editor.org/rfc/rfc2119)
- [RFC 8174: Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words](https://www.rfc-editor.org/rfc/rfc8174)

## Appendix A: Conformance Tests

Implementations MUST pass all tests in categories 1-2 for Core conformance,
categories 1-5 for Full conformance, and categories 1-6 for Interop
conformance.

### Category 1: Event Structure (10 tests)

| ID | Test | Requirement |
|----|------|-------------|
| E01 | Kind 1111 rumor has correct structure | `kind: 1111`, `pubkey` set, `tags` array, `content` string |
| E02 | Rumor has no valid signature | `sig` is empty or absent |
| E03 | Recipient `p` tags include role marker | `["p", pubkey, relay, "to"/"cc"]` |
| E04 | Subject tag is present | `["subject", "..."]` |
| E05 | Content-type tag omitted for `text/plain` | Default content type is not tagged |
| E06 | Content-type tag present for non-default types | `["content-type", "text/markdown"]` |
| E07 | Reply tag references parent event | `["reply", eventId, relayHint]` |
| E08 | Thread tag references root event | `["thread", eventId, relayHint]` |
| E09 | Attachment tags include all required fields | `["attachment", hash, filename, mime, size]` |
| E10 | Tag elements are all strings | Numeric sizes serialized as decimal strings |

### Category 2: Encryption (12 tests)

| ID | Test | Requirement |
|----|------|-------------|
| C01 | Seal is correctly formed | `kind: 13`, `pubkey: sender`, empty `tags` |
| C02 | Seal content decrypts with NIP-44 | ECDH(recipient, sender) yields valid JSON |
| C03 | Seal signature is valid | Schnorr signature verifies against seal.pubkey |
| C04 | Gift wrap is correctly formed | `kind: 1059`, ephemeral pubkey, `["p", recipient]` |
| C05 | Gift wrap uses ephemeral key | wrap.pubkey differs from sender pubkey |
| C06 | Gift wrap content decrypts with NIP-44 | ECDH(recipient, ephemeral) yields valid seal |
| C07 | Gift wrap signature is valid | Schnorr signature verifies against ephemeral pubkey |
| C08 | Round-trip preserves content | wrap(rumor) then unwrap yields identical rumor |
| C09 | Seal timestamp is randomized | seal.created_at differs from rumor.created_at |
| C10 | Wrap timestamp is randomized | wrap.created_at differs from seal.created_at |
| C11 | Different ephemeral keys per recipient | Same rumor, two recipients, different wrap pubkeys |
| C12 | Different ciphertext per wrap | Same rumor wrapped twice yields different content |

### Category 3: Anti-Spam (7 tests)

| ID | Test | Requirement |
|----|------|-------------|
| S01 | Contact sender classifies as tier 0 | Sender in kind 3 follow list |
| S02 | NIP-05 sender classifies as tier 1 | Sender has valid NIP-05 |
| S03 | Sufficient PoW classifies as tier 2 | Event PoW >= policy threshold |
| S04 | Valid Cashu P2PK token classifies as tier 3 | Token amount >= threshold, P2PK locked |
| S05 | No qualifying signal classifies as tier 5 | Unknown sender, no signals |
| S06 | Tier evaluation uses highest-priority match | Contact + PoW yields tier 0, not tier 2 |
| S07 | Bearer tokens (no P2PK) are rejected | Non-P2PK Cashu token yields tier 5 |

### Category 4: Mailbox State (7 tests)

| ID | Test | Requirement |
|----|------|-------------|
| M01 | Read state is append-only | markRead adds to reads set |
| M02 | markRead is idempotent | Same ID twice yields same state |
| M03 | Read state cannot be reverted | No operation removes from reads set |
| M04 | State serializes to kind 10099 tags | Correct tag format for reads, flags, folders |
| M05 | State deserializes from tags | Tag parsing round-trips correctly |
| M06 | Merge: G-Set union for reads | Merged reads is union of both sets |
| M07 | Merge: flags from both states preserved | Union of flag arrays per event ID |

### Category 5: Threading (6 tests)

| ID | Test | Requirement |
|----|------|-------------|
| T01 | Root message has no parent | No reply/thread tags yields root node |
| T02 | Reply links to parent | reply tag yields child relationship |
| T03 | Thread tag points to root | Same thread tag across conversation |
| T04 | Thread tree is correctly built | Valid parent-child relationships |
| T05 | Siblings sorted by created_at | Chronological order within each level |
| T06 | Orphaned replies handled | Reply to unknown parent treated as root |

### Category 6: Interoperability (4 tests)

| ID | Test | Requirement |
|----|------|-------------|
| I01 | Cross-implementation wrap/unwrap (A wraps, B unwraps) | Round-trip succeeds |
| I02 | Cross-implementation wrap/unwrap (B wraps, A unwraps) | Round-trip succeeds |
| I03 | Malformed kind 1059 events rejected gracefully | No crash on invalid input |
| I04 | Unknown tags ignored gracefully | Extra tags do not cause errors |

### Conformance Levels

| Level | Requirements | Meaning |
|-------|-------------|---------|
| **Core** | Categories 1-2 (E01-C12) | Can send and receive encrypted NOSTR Mail |
| **Full** | Categories 1-5 (E01-T06) | Full protocol support |
| **Interop** | Categories 1-6 (E01-I04) | Verified cross-implementation compatibility |

## Appendix B: Implementation Notes

### Recommended Libraries

| Language | NIP-44/NIP-59 | Cashu | Blossom |
|----------|---------------|-------|---------|
| TypeScript | nostr-tools (nip44, nip59 modules) | @cashu/cashu-ts | fetch API |
| Go | go-nostr (nip44, nip59 packages) | gonuts | net/http |
| Rust | nostr crate (nip44, nip59 modules) | cdk (Cashu Dev Kit) | reqwest |
| Python | pynostr or nostr-sdk (via FFI) | cashu (nutshell) | httpx |

### Common Pitfalls

1. **Signing the rumor.** Kind 1111 and kind 1112 events MUST NOT be signed. The
   `sig` field must be empty or absent. Signing the rumor eliminates
   deniability.

2. **Reusing ephemeral keys.** Each gift wrap MUST use a unique ephemeral key.
   Reusing keys across recipients allows linking wraps to the same sender.

3. **Non-string tag elements.** All tag array elements must be JSON strings.
   A common mistake in dynamically typed languages is serializing numeric
   values (file sizes, PoW bits, Cashu amounts) as JSON numbers instead of
   strings.

4. **Omitting intermediate tag elements.** When a `p` tag has a role but no
   relay hint, the relay hint position MUST contain an empty string `""`. Do
   not omit it, as this shifts the role to the wrong index.

5. **Using Math.random() for cryptographic values.** All random values
   (ephemeral keys, timestamps, encryption keys, IVs) MUST use a CSPRNG.

6. **Assuming serialization order.** Rumor JSON serialization order is not
   specified. Parse decrypted JSON by key name, not by position. Do not
   compare ciphertext across implementations.

7. **Forgetting the self-copy.** Senders MUST gift-wrap a copy to themselves.
   Without this, sent messages cannot be recovered.

8. **P2PK prefix.** When constructing the P2PK spending condition for Cashu
   tokens, always prepend `0x02` to the NOSTR x-only pubkey. Do not attempt
   to compute actual y-coordinate parity.
