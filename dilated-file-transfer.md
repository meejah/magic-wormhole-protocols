# Dilated File-Transfer Protocol

This version of the file-transfer protocol is a complete replacement for the original file-transfer protocol (which we now refer to as "classic").

Both sides must support and use Dilation (see `dilation-protocol.md`).

Any all-caps words ("MAY", "MUST", etc) follow RFC2119 conventions.

NOTE: there are several open questions / discussion points, some with corresponding "XXX" comments inline.


## Overview and Features

Dilated File Transfer is a flexible, session-based approach to file transfer allowing either side to offer files (or groups of file) to send while the other side may accept or reject each offer.
Either side MAY terminate the transfer session (by closing the wormhole)
Either side MAY select a simpler one-way mode, similar to the classic protocol.
An extension mechanism allows for future (optional) features.

Files are offered and sent individually, with no dependency on zip or other archive formats.

Metadata is included in the offers to allow the receiver to decide if they want that file before the transfer begins.

"Offers" generally correspond to what a user might select; a single-file offer is possible but so is a directory.
In both cases, they are treated as "an offer" even though a directory may consist of dozens or more individual files.

Filenames are relative paths.
When sending individual files, this will simply be the filename portion (with no leading paths).
For a series of files in a directory (i.e. if a directory was selected to send) paths will be relative to that directory (starting with the directory itself).
(XXX see "file naming" in discussion)


## Version Negotiation

There is an existing file-transfer protocol which does not use Dilation (called "classic" in this document).
Clients supporting newer versions of file-transfer (i.e. the one in this document) SHOULD offer backwards compatibility where possible.

In the core mailbox protocol, applications can indicate version information, via `app_versions`.
The existing file-transfer protocol doesn't use this feature so the version information is empty (indicating "classic").
This new Dilated File Transfer MUST include version information:

```json
{
    "transfer": {
        "mode": "{send|receive|connect}",
        "features": ["core0"],
        "permission": "{ask|yes}"
    }
}
```

**Rejected idea**: having a `version` number that increases.
Per RFC 9170, it can be the case that a protocol can "ossify" or lost its flexibility, including when using "highest common version" sorts of negotiation.
Multiple extension points (e.g. both "version" and "features") can cause confusion; including both was rejected after considering the question, "when would `version` be incremented instead of using a `feature`?"

The `mode` key indicates the desired mode of that peer.
It has one of three values:
* `"send"`: the peer will only send files (similar to classic transfer protocol)
* `"receive"`: the peer only receive files (the flip side of the above)
* `"connect"`: the peer will send and receive zero or more files before closing the session

Note that `send` and `receive` above will still use Dilation as all clients supporting this protocol must.
If a peer sends no version information at all, it will be using the classic protocol (and is thus using Transit and not Dilation for the peer-to-peer connection).

The `"features"` key is a list of message-formats / features understood by the peer.
This allows for existing messages to be extended, or for new message types to be added.
Peers MUST _accept_ messages for any features they support.
Peers MUST only send messages for features in the other side's list.
Only one feature exists currently: `"core0"` (and it MUST be supported).
Peers MUST tolerate the existence of unknown values in the `"features"` list.

   (XXX might make the Python implementation randomly add an unknown one, 10% of the time?)

See "Example of Protocol Expansion" below for discussion about adding new attributes or capabilities or evolving existing ones.

The `"permission"` key specifies how to proceed with offers.
Using mode `ask` tells the sender to pause after transmitting the first metadata message and await an answer from the peer before streaming data.
Using mode `yes` means the peer will accept all incoming transfers so the sender should not pause after the metadata (and instead immediately start sending data messages).
This cuts down latency for "one-way" transfers (see "Discussion")


## Protocol Details

See the Dilation document for details on the Dilation setup procedure.
Once a Dilation-supporting connection is open, we will have a "control" subchannel (subchannel #0).
Either peer can also open additional subchannels.

All control-channel messages are encoded using `msgpack` (rejected idea: JSON, because it lacks integers and binary types).

Control-channel message formats are described using Python (and Haskell) pseudo-code to illustrate the exact data types involved.

All control-channel messages contain a "kind" field describing the type of message.

**Rejected idea**: Version message, because we already do version negotiation via mailbox features.

**Rejected idea**: Offer/Answer messages via the control channel: we need to open a subchannel anyway; and the subchannel-IDs are not intended to be part of the public Dilation API.


### Control Channel Messages

Each side MAY send a free-form text message at any time.
These messages look like:

```python
class Message:
    kind: "text"
    message: str     # unicode string
```

All control-channel messages MUST be msgpack-encoded and include at least a `"kind"` field.
Future extensions to the protocol may add additional control-channel messages (thus, any unknown control-channel messages MUST be ignored).
To be clear, an unknown control-channel `"kind"` MUST NOT be treated as a protocol error.


### Making an Offer

Either side MAY send any number of Offer messages at any time after the connection is set up.
If the other peer specified `"mode": "send"` then this peer MUST NOT make any offers.

To make an offer the peer opens a subchannel.
Recall from the Dilation specification that subchannels are _record_ pipes (not simple byte-streams).
That is, a subchannel transmits a series of complete (framed) messages (up to ~4GiB in size).

For this protocol, each record on the subchannel uses the first byte to indicate the kind of message; the remaining bytes in any message are kind-dependant.

The following kinds of messages exist (indicated by the first byte):
* 0x00: reserved / unused
* 0x01: msgpack-encoded `FileOffer` message
* 0x02: msgpack-encoded `DirectoryOffer` message
* 0x03: msgpack-encoded `OfferAccept` message
* 0x04: msgpack-encoded `OfferReejct` message
* 0x05: file data bytes

All other byte values are reserved for future use and MUST NOT be used.
(The "features" mechanism can be used to experiment with new sorts of messages, as ultimately a new feature needs to be described in this protocol documnet and that description may specify more "kinds" of message).

The first message sent on the new subchannel MUST be either `FileOffer` or `DirectoryOffer`.

To offer a single file (with message kind `1`):

```python
class FileOffer:
    filename: str    # unicode relative pathname
    timestamp: int   # Unix timestamp (seconds since the epoch in GMT)
    bytes: int       # total number of bytes in the file
```

To offer a directory tree of many files (with message kind `2`):
```python
class DirectoryOffer:
    base: str          # unicode path segment of the root directory (i.e. what the user selected)
    size: int         # total number of bytes in _all_ files
    files: list[str]   # a list containing relative paths for each file
```

The filenames in the `"files"` list are unicode relative paths (relative to the `"base"` from the `DirectoryOffer` and NOT including that part.

For example:

```python
DirectoryOffer(
    base="project",
    size: 165,
    files=["README", "src/hello.py"]
    ]
)
```

This indicates an offer to send two files, one in `"project/README"` and the other in `"project/src/hello.py"`.
A client can consider the "base" name as suggestion, of course.
On the flip side, a privacy-conscious sending application could offer to randomize the name when sending (or at least use something other than the on-filesystem name).

In Haskell, this might look like:

```haskell
import Path (Dir, File, Rel, Path)
import Data.Time (UTCTime)
import Numeric.Natural (Natural)

data Offer
  = FileOffer {
      fname :: Path Rel File,
      timestamp :: UTCTime,
      size :: Natural }
  | DirectoryOffer{
      base :: Path Rel Dir,
      total_size :: Natural,
      files :: [Path Rel File] }
```


What happens next depends on the mode of the peer.

If the peer has `"mode": "yes"` then this peer MUST immediately start sending content messages (see below).

If the peer has `"mode": "ask"` then this peer MUST NOT send any more messages and instead await an incoming message.

Note that a UI treatment can still have a list with multiple offers in it; this protocol is spoken per-subchannel so another offer would be on a separate subchannel.

In `"mode": "ask"`, the incoming message MUST be either `OfferAccept` or `OfferReject`.
These are indicated by the kind byte of that message being `3` or `4` (see list above).

```python
class OfferReject:
    reason: str      # unicode string describing why the offer is rejected
```

Accept messages are blank (that is, they are a single byte: `4`).

```python
class OfferAccept:
    pass
```

Or in Haskell:

```haskell
data Answer = OfferAccept | OfferReject(Text)
```

When the offering side gets an `OfferReject` message, the subchannel MUST be immediately closed (by the offering side).
The offering side SHOULD show the "reason" string to the user.

When the offering side gets an `OfferAccept` message it begins streaming the file over the already-opened subchannel.
When completed, the subchannel is closed.

That is, the offering side always initiates the open and close of the corresponding subchannel.

Messages of kind `5` ("file data bytes") consist solely of file data.
A single data message MUST NOT exceed 65536 (65KiB) inculding the single byte for "kind".
Applications are free to choose how to fragment the file data so long as no single message is bigger than 65536 bytes.
A good default to choose in 2024 is 16KiB (2^14 - 1 payload bytes)

When sending a `DirectoryOffer` each individual file is preceeded by a `FileOffer` message.
However the rules about "maybe wait for reply" no longer exist; that is, all file data SHOULD be immediately sent.
The receiving side MUST NOT send a reply message (`OfferReject` or `OfferAccept`) in this case.

See examples down below, after "Discussion".


## Discussion and Open Questions

* streaming data

There is no "finished" message. Maybe there should be? (e.g. the receiving side sends back a hash of the file to confirm it received it properly?)

Does "re-using" the `FileOffer` as a kind of "header" when streaming `DirectoryOffer` contents make sense?
We need _something_ to indicate the next file etc

Do the limits on message size make sense? Should "65KiB" be much smaller, potentially?
(Given that network conditions etc vary a lot, I think it makes sense for the _spec_ to be somewhat flexible here and "65k" doesn't seem very onerous for most modern devices / computers)


* compression

It may make sense to do compression of files.
See "Protocol Expansion Exercises" for more discussion.

Preliminary conclusion: no compression in this version.


## File Naming Example

Given a hypothetical directory tree:

* /home/
  * meejah/
    * grumpy-cat.jpeg
    * homework-draft2-final.docx
    * project/
      * local-network.dia
      * traffic.pcap
      * README
      * src/
        * hello.py

As spec'd above, if the human selects `/home/meejah/project/src/hello.py` then it should be sent as `hello.py`.
However if they select `/home/meejah/project/` then there should be a Directory Offer offers like:

```python
DirectoryOffer(
    base="project",
    size=4444,
    files=[
        "local-network.dia",
        "traffic.pcap",
        "README",
        "src/hello.py"
    ]
)
```

The sending client UX could offer to change the base name to something else.
The receiving client could choose to "suggest" the name, or simply use it, or anything else deemed appropriate.


## Protocol Expansion Exercises

Here we present several scenarios for different kinds of protocol expansion.
The point here is to discuss the _kinds_ of expansions that might happen.
These examples here ARE NOT part of the spec and SHOULD NOT be implemented.

That said, we believe they're realistic features that _could_ make sense as future protocol expansions.

Per advice in RFC 9170, it may be good to have 2 or more features present in the "first version" of the protocol to ensure this extension mechanism gets exercised.
(XXX put "compression" in here? so then there's an optional feature?)


### Thumbnails

Let us suppose that at some point in the future we decide to add `thumbnail: bytes` to the `Offer` messages.
It is reasonable to imagine that some clients may not make use of this feature at all (e.g. CLI programs) and so work and bandwidth can be saved by not producing and sending them.

This becomes a new `"feature"` in the protocol.
That is, the version information is upgraded to allow `"features": ["basic", "thumbnails"]`.

Peers that do not understand (or do not _want_) thumbnails do not include that in their `"features"` list.
So, according to the protocol, these peers should never receive anything related to thumbnails.
Only if both peers include `"features": ["basic", "thumbnails"]` will they receive thumbnail-related information.

The thumbnail feature itself could be implemented by expanding the `Offer` message:

```python
class Offer:
    filename: str
    timestamp: int
    bytes: int
    thumbnail: bytes  # introduced in "thumbnail" feature; PNG data
```

A new peer speaking to an old peer will never see `thumbnail` in the Offers, because the old peer sent `"formats": ["basic"]` so the new peer knows not to inclue that attribute (and the old peer won't ever send it).

Two new peers speaking will both send `"formats": ["basic", "thumbnails"]` and so will both include (and know how to interpret) `"thumbnail"` attributes on `Offers`.

Additionally, a new peer that _doesn't want_ to see `"thumbnail"` data (e.g. it's a CLI client) can simply not include `"thumbnail"` in their `"formats"` list even if their protocol implementation knows about it.


### Finer Grained Permissions

What if we decide we want to expand the "ask" behavior to sub-items in a Directory.

As this affects the behavior of both the sender (who now has to wait more often) and the receiver (who now has to send a new message) this means an overall protocol version bump.

(XXX _does_ it though? I think we could use `"features"` here too if we wanted...)

So, this means that `"version": 2` would be the newest version.
Any peer that sends a version lower than this (i.e. existing `1`) will not send any fine-grained information (or "yes" messages).
Any peer who sees the other side at a version lower than `2` thus cannot use this behavior and has to prtend to be a version `1` peer.

If both peers send `2` then they can both use the new behavior (still using the overall `"yes"` versus `"ask"` switch that exists now, probably).


### Compression

Suppose that we decide to introduce compression.

(XXX again, probably just "features"?)


### Big Change

What is a sort of change that might actually _require_ us to use the `"version": 2` lever?


### How to Represent Overall Version

It can be useful to send a "list of versions" you support even if the ultimate outcome of a "version negotiation" is a single scalar (of "maximum version").

Something to do with being able to release (and then revoke) particular (possibly "experimental") versions.

There may be an example in the TLS history surrounding this. (`RFC 9170 <https://datatracker.ietf.org/doc/html/rfc9170>`_ discusses this, or at least the concept of "greasing" a protocol's extension points)

This means we might want `"version": [1, 2]` for example instead of `"version": 1` or `"version": 2` alone.

XXX expand, find TLS example


## Example: one-way transfer

Alice has two computers: `laptop` and `desktop`.
Alice wishes to transfer some number of files from `laptop` to `desktop`.

Alice initiates a `wormhole receive --yes` on the `desktop` machine, indicating that it should receive multiple files and not ask permission (writing each one immediately).
This program prints a code and waits for protocol setup.

Alice uses a GUI program on `laptop` that allows drag-and-drop sending of files.
In this program, she enters the code that `desktop` printed out.

Once the Dilated connection is establed, Alice can drop any number of files into the GUI application and they will be immediately written on the `desktop` without interaction.

Speaking this protocol, the `desktop` (receive-only CLI) peer sends version information:

```json
{
    "transfer": {
        "version": 1,
        "mode": "receive",
        "features": ["basic"],
        "permission": "yes"
    }
}
```

The `laptop` peer then knows to not pause on its subchannels to await permission (`"permission": "yes"`).
For each file that Alice drops, the `laptop` peer:
* opens a subchannel
* sends a `FileOffer` / kind=`1` record
* immediately starts sending data (via kind=`5` records)
* closes the subchannel (when all data is sent)

On the `desktop` peer, the program patiently waits for subchannels to open.
When a subchannel opens, it:
* reads the first record and finds a `FileOffer` and opens a local file for writing
* reads subsequent data records, writing them to the open file
* notices the subchannel close
* double-checks that the corrent number of payload bytes were received
   * XXX: return a checksum ack? (this might be more in line with Waterken principals so the sending side knows to delete state relating to this file ... but arguably with Dilation it "knows" that file made it?)
* closes the file

When the GUI application finishes (e.g. Alice closes it) the mailbox is closed.
The `desktop` peer notices this and exits.


## Example: multi-directional transfer session

Alice and Bob are on a video-call together.
They are collaborating and wish to share at least one file.

Both open a GUI wormhole application.
Alice opens hers first, clicking "connect" to generate a code.
She tells Bob the code, and he enters it in his GUI.

A Dilated channel is established, and both GUIs indicate they are ready to receive and/or send files.

As Alice drops files into her GUI, Bob's side waits for confirmation.
Several files could be in this state.
Whenever Bob clicks "accept" on a file, his client answers with an `OfferAccept` message and Alice's client starts sending data records (the content of the file).
If Bob were to click "reject" then his client would answer with an `OfferReject` and Alice's client would close the subchannel.
XXX what if Bob is bored and clicks "cancel" on a file?

Alice and Bob may exchange severl files at different times in either direction.
As they wrap up the call, Bob close his GUI client which closes the mailbox (and Dilated connection).
Alice's client sees the mailbox close.
Alice's GUI tells her that Bob is done and finishes the session; she can no longer drop or add files.
