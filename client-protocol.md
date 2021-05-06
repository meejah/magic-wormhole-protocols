# Client-to-Client Protocol

Wormhole clients do not talk directly to each other (at least at first): they
only connect directly to the Rendezvous Server. They ask this server to
convey messages to the other client (via the `add` command and the `message`
response).
The goal of this protocol is to establish an authenticated and encrypted
communication channel over which data messages can be exchanged. It is not
suited for larger data transfers. Instead, the channel should be used to coordinate
the negotiation of a better suited communication channel. This is what the
[transit protocol](transit.md) is for (the actual negotiation is done by the
application-specific protocol, for example [file transfer](file-transfer-protocol.md).

This document explains the format of these client-to-client messages. Each
such message contains a "phase" string, and a hex-encoded binary "body":

```json
{
  "phase": <string or number>,
  "body": "<hex bytes>",
}
```

From now on, all mentioned JSON messages will be from within the `body` field.

Any phase which is purely numeric (`^\d+$`) is reserved for encrypted
application data. The Rendezvous server may deliver these messages multiple
times, or out-of-order, but the wormhole client will deliver the
corresponding decrypted data to the application in strict numeric order. All
other (non-numeric) phases are reserved for the Wormhole client itself.
Clients will ignore any phase they do not recognize.
The order of the wormhole phases are: `pake`, `version`, numerical.

## Pake

Immediately upon opening the mailbox, clients send the `pake` phase, which
contains the binary SPAKE2 message (the one computed as `X+M*pw` or
`Y+N*pw`):

```json
{
  "pake_v1": <hex-encoded pake message>,
}
```

Upon receiving their peer's `pake` phase, clients compute and remember the
shared key. They derive a "verifier", which is a subkey of the shared key
generated with `wormhole:verifier` as purpose): applications can display
this to users who want additional assurance (by manually comparing the values
from both sides: they ought to be identical). This protects against the threat
where a man in the middle attacker correctly guesses the password.

From this point on, all messages are encrypted using a NaCl `SecretBox` or some
semantic equivalent. The key is derived from the shared key using the following
purpose (as pseudocode-fstring): `f"wormhole:phase:{sha256(side)}{sha256(phase)}"`.
The key derivation function is HKDF-SHA256, using the shared PAKE key as the secret.
A random nonce is used, and nonce and ciphertext are concatenated. Their hex
encoding is the content of the `body` field.

## Version exchange

In the `version` phase, the clients exchange information about themselves in
order to do feature negotiation. Unknown keys and values must be ignored.

The optional `abilities` key allows the two Wormhole instances
to signal their ability to do other things (like "dilate" the wormhole).
It defaults to the empty list. Both sides intersect their abilities with their
peer's ones, in order to determine wich protocol extensions will be used. An
ability might define more keys in the dict to exchange more detailed information
about that feature apart from "I support it". Currently specified abilities are:
`dilation-v1`, `seeds-v1`.

The version data will also include an `app_versions` key which contains a
dictionary of metadata provided by the application, allowing apps to perform
similar negotiation. Its value is determined by the application-specific protocol,
for example [file transfer](file-transfer-protocol.md).

```json
{
  "abilities": [],
  "app_versions": {},
}
```

As this is the first encrypted message, it also serves as a test to check if
the encryption worked or failed (i.e. if the user typed the password correctly
and no attackers are involved).
The client knows the supposed shared key, but has not yet seen
evidence that the peer knows it too. When the first peer message arrives, it will
be decrypted: we use authenticated encryption (`nacl.SecretBox`), so if this
decryption succeeds, then we're confident that *somebody* used the same
wormhole code as us. This event pushes the client mood from "lonely" to
"happy".

If any message cannot be successfully decrypted, the mood is set to "scary",
and the wormhole is closed, the nameplate/mailbox
will be released, and the WebSocket connection will be dropped.

## Application-specific

Now the client connection is fully set up, and the application specific messages
(those with numeric phases) may be exchanged.

## Wormhole Seeds

Once to clients ever connected to each other, they now have a shared secret. This can
be used to establish a new Wormhole connection without involving human entering codes.
The code to be used can be derived from the previous session's key. No nameplate is needed,
both sides can simply claim and connect to the same mailbox as last time. Some additional
data needs to be exchanged and stored in order to allow for a good user experience.

Cryptographically, this is pretty streightforward. If two seeds-enabled clients connect
to each other, they may find each other again at `mailbox' = sha256(mailbox)`, using
`code = hex(derive(key, "wormhole:seed"))` as "code". Instead of using nameplates in the
client-server protocol, they can directly `open` the mailbox and start the client to client
protocol as usual.

Clients that want to support session resumption declare their abilitiy using the `seeds-v1`
value during the `versions` phase. Additionally, they add a `seed` key that looks like this to
their versions message:

```json
{
  "abilities": [ "seeds-v1" ],
  "app_versions": {},
  "seed": {
    "uuid": <UUID-v4>,
    "name": <string>,
  },
}
```

Wormhole seeds are used by the user saying "I want to connect to this session", instead of "I
want to connect to this code". Thus, there needs to be a mapping for the user to tell which
session to resume. For this reason, every client generates a random, persistent
[UUID](https://en.wikipedia.org/wiki/Universally_unique_identifier). Every time two clients
connect, they update the seed accordingly. The UUIDs can be used to deduplicate sessions.
Additionally, the optional `name` attribute may help giving human-readable names to sessions.
A good default would be the user's name. But there must be a way to customize this, in the case
a user has multiple devices and thus multiple sessions.

Because UUIDs may change, sessions may die and some connections are oneshots. An expiry time of
12 months is recommended. It is up to the clients if they want to make pairings explicit
