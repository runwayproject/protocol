# Runway Protocol
# 1. Overview
This document specifies a messaging protocol that provides end to end encrypted communication using the Message Layer Security (MLS) protocol. 

Runway is designed such that:
- Servers only act as relays and coordinators. They do not store conversation state.
- Conversation structure is client owned.
- Servers cannot determine sender identity (via sealed sender), conversation membershsip or conversation type.
- One-to-one and group messaging are indistinguishable at the cryptographic and protocol level, except for unavoidable metadata such as delivery fan-out (see 8.1)

# 2. Roles and Trust Model
## 2.1 Roles
Clients:
- Must generate and manage all cryptographic keys locally.
- Must maintain MLS group state locally
- Must validate message authenticity and integrity locally
- Must handle message decryption locally

Clients are assumed to be compromised at the endpoint level, MLS provides forward secrecy and post-compromise security to mitigate this risk.
## 2.2 Servers
Servers:
- Treat all messages (or any other info clients may broadcast) as opaque blobs
- Must not maintain group membership or room state
- May queue encrypted blobs if the client is offline; the client can fetch queued blobs when they come online
- May observe unavoidable metadata (timing, size, recipient)
- May use anti-abuse measures (e.g. rate limiting) without compromising privacy guarantees

Servers are assumed to be compromised and malicious. The protocol is designed to minimize the information servers can learn about conversations and participants.

# 3. Cryptography
## 3.1 MLS Usage
- All conversations, including one-to-one chats, are implemented as MLS groups. This provides a consistent security model and feature set across all conversation types.
- One on one conversations use 2 member MLS groups
- Group conversations use MLS groups with 3 or more members
- Clients must process MLS commits and epochs according to the MLS specification
- Groups should not exceed 1000 members. Beyond this size, epoch conflicts, fan-out metadata and other issues may rise significantly. Implementations can enforce a lower or higher limit.

## 3.2 MLS Guarantees
MLS provides:
- End to end encryption: Only group members can decrypt messages
- Forward secrecy: Compromise of current keys does not compromise past messages
- Post-compromise security: After a compromise, future messages can remain secure

The protocol relies on MLS for the majority of cryptographic guarantees. The transport layer may use standard public key encryption for sealed sender and routing.

## 3.3 Sealed Sender Construction
Runway achieves sealed sender structurally. The transport layer contains no sender identity field whatsoever. The server receives only the recipient RID, the ciphertext, and a timestamp. Sender identity is discovered exclusively by the recipient through MLS. When the recipient decrypts the ciphertext, the MLS signature on the inner message authenticates the sender as a member of the shared group. The server cannot determine sender identity because the server cannot decrypt the blob.

### 3.3.1 Transport Authentication vs Message Authentication
The runway-auth-v1 scheme is used exclusively for authenticating transport-layer requests that require server authorization: RID issuance, RID rotation, and queued blob retrieval. These operations require the server to verify the client owns the credential associated with the RID. Blob submission (PutBlob) is intentionally unauthenticated at the transport layer, requiring auth would link a sender identity to the relay server and break the sealed sender guarantee. Message-level authenticity is handled entirely by MLS.

# 4. Message Structure
## 4.1 Overview
The protocol does not define any distinction between:
- One-to-one conversations
- Group conversations
- Channels

All are MLS groups with different numbers of members. The message structure is designed to be opaque to the server, with all relevant information encoded in the MLS ciphertext.    
## 4.2 Group Identity
MLS group IDs must be encrypted inside blobs when transmitted. 

Group IDs are transmitted exclusively inside MLS ciphertext and never appear in the transport envelope. The server cannot extract, correlate, or identify group IDs from observed blobs. Two blobs belonging to the same group are unlinkable at the transport layer.

# 5. Transport Model
## 5.1 Encrypted Blob
The sole unit of transport is the encrypted blob. It contains the recipient ID and the ciphertext. The server relays blobs to the intended recipient(s) without inspecting their contents.

Recipient IDs are opaque, server-assigned routing identifiers that MAY rotate periodically. The protocol does not assume they are stable or user-meaningful

The server must not be able to determine:
- Sender identity (via sealed sender)
- Group membership
- Message type
- Obviously, message content

### 5.1.1 Blob Structure
Each encrypted blob contains the MLS ciphertext and the recipient RID. The server sees the recipient RID and the size of the blob, but cannot decrypt or inspect the ciphertext. The sender's MLS identity is not present at the transport layer at all, it exists only within the MLS ciphertext, where it is visible to the recipient upon decryption but invisible to the server. Recipients authenticate the sender using MLS signatures, independent of the RID.

### 5.1.2 Blob Wire Format
Blobs are encoded using CBOR and framed with a 4-byte big-endian length prefix indicating the size of the encoded payload in bytes.
The structure is as follows:

```rust
pub struct EncryptedBlob {
    pub recipient_rid: String,
    pub ciphertext: Vec<u8>,
    pub created_at_unix_ms: u64,
}
```

```rust
pub fn write_framed<W: Write>(writer: &mut W, payload: &[u8]) -> Result<()> {
    let len: u32 = payload // get the payload length
        .len()
        .try_into()
        .context("payload too large to frame")?;

    writer // write the length prefix
        .write_all(&len.to_be_bytes())
        .context("writing frame length failed")?;

    writer // write the contents
        .write_all(payload)
        .context("writing frame payload failed")?;

    writer.flush().context("flushing framed payload failed")?;
    Ok(())
}
```
The maximum permitted frame length is enforced by the receiver. Frames exceeding the configured maximum must be rejected. The created_at_unix_ms field is set client-side and must not be trusted by the server for any security-relevant purpose.

## 5.2 Delivery
- Clients upload encrypted blobs addressed to recipient IDs. The server relays these blobs to the intended recipients.
- If a recipient is offline, the server may queue the blob for later delivery. The client can fetch queued blobs when they come online.
- Blobs are decrypted locally by the client
## 5.3 Authentication and Integrity
- Recipients authenticate senders using MLS signatures. The server cannot forge messages or impersonate senders due to the use of MLS signatures and sealed sender.
- Sender authenticity must be verified client-side
MLS signatures authenticate the sender to group members only; sender identity is intentionally visible to recipients while remaining hidden from the server.

Requests for receiver ID issuance, rotation, or message retrieval MUST be authenticated by the client via a signature over the request using the private key associated with the client’s registered MLS credential. This authentication authorizes routing capabilities only and does not convey message-level authenticity or group membership.
## 5.4 Sender Anonymity Model
Runway supports sealed sender semantics, where the sender’s MLS identity is cryptographically hidden from the server.

The protocol does not guarantee network-layer anonymity. Malicious servers may observe source IP addresses unless clients use anonymizing transports (proxies or VPNs).

## 5.5 Ephemeral Receiver IDs
All message delivery occurs via ephemeral receiver IDs (RIDs), which are temporary routing identifiers issued by the server and used in place of persistent user addresses. Clients are responsible for requesting and rotating their RIDs, which may expire or change at any time. RIDs act purely as opaque delivery endpoints: the server routes blobs to the appropriate RID without learning the user’s identity, group membership, or conversation type. Authorized peers receive updated RIDs through MLS application messages or pre-shared tokens, ensuring that message content and recipient identity remain cryptographically hidden from the server. By using ephemeral RIDs, the protocol enforces strong privacy and limits the metadata exposure associated with message routing.

RIDs are expressed in <rid>@server.org format universally, as defined in 10.1.

RIDs are merely routing handles the server internally uses, not cryptographic identities. They do not carry any inherent information about the user or their conversations, and their rotation is a key mechanism for maintaining privacy and preventing correlation by the server.

MLS updates, commits, and epochs are independent of RID rotation, clients simply send blobs to the current valid RID of each member.

### 5.5.1 RID Issuance and Rotation
#### Initial Issuance:
A brand new client has no MLS credential and no RID. Bootstrap proceeds in the following order:

1. Client generates its MLS credential locally
2. Client sends an IssueRid request to the relay server, authenticated with the newly generated credential
3. Server issues an ephemeral RID bound to that credential fingerprint
4. Client includes this RID in its contact token for out-of-band sharing

#### Rotation:
When a client rotates its RID, it sends a RotateRid request authenticated with its existing credential. The server issues a new RID and invalidates the old one. The client must notify all peers of the new RID via MLS application messages within any shared groups. Peers update their local routing state upon receiving the notification.

RID rotation does not affect MLS group state or epoch. Only the transport layer address changes. Clients should rotate their RID periodically to limit long-term correlation by the server or passive observers.

#### Expiry:
RIDs carry an expiry timestamp issued by the server. Clients are responsible for renewing their RID before expiry. A client that allows its RID to expire will be unreachable until it issues a new one and notifies peers.

## 5.6 Initial Contact and Contact Tokens
Establishing initial communication with a user requires explicit sharing of information. A contact token is generated by the recipient and shared out-of-band via a link, QR code, or similar mechanism. It contains the minimum information necessary for a peer to initiate first contact:

- Initial ephemeral RID (required): the recipient's current routing address on their relay server
- Keyserver handle and domain (if a keyserver is used): allows the adder to fetch a KeyPackage without synchronous interaction. otherwise, it first falls back to the KeyPackage that is included in the contact token, and if that is not present, it falls back to the interactive KeyPackageRequest flow.

The token format is as follows: 
`runway::v1::<rid>@<relayserver>::<handle>@<keyserver>::<keypackage>`
The keyserver and keypackage components are optional. The adder can choose to include a keyserver handle for asynchronous adds, or include a KeyPackage directly in the token for an asynchronous add without a keyserver.

When a keyserver handle is present, the contact token does not need to embed a KeyPackage directly, as one can be fetched on demand. When no keyserver handle is present, the adder should fallback to the KeyPackage that is included in the contact token, and if that is not present, it falls back to the interactive KeyPackageRequest flow.

Possession of a contact token allows the receiving client to construct a valid MLS Welcome message and begin sending messages. The server does not learn the identity of either participant or the content of any messages. First contact is a deliberate action controlled by the users themselves. No automatic discovery by username or any other identifier is provided, as this would reveal metadata and compromise privacy.

# 6. Invites and group join
## 6.1 Invites
Group joins are standard MLS. An existing group member generates a GroupInfo object containing the group context and the public key material necessary for the new member to derive shared secrets. This GroupInfo is encoded in an invite link or token and shared with the new member out-of-band.

The new member uses the GroupInfo to construct an MLS Welcome message, which allows them to join the current epoch directly. 

The server cannot determine the new member's identity and cannot cryptographically verify that a group membership change has occurred, though metadata such as message size and fan-out may indicate that a group update took place.

## 6.2 MLS Group Join via Ephemeral Routing
To join an existing MLS group, a new member must receive a GroupInfo object securely, typically through a pre-shared invite token or link that includes the initial ephemeral receiver ID. All message delivery, including the initial GroupInfo and the Welcome, occurs via ephemeral receiver IDs encapsulated in opaque blobs relayed by the server. The server cannot decrypt group content, determine group membership, or identify the new member; it only observes delivery fan-out and timing. Subsequent updates, including receiver ID rotations, are communicated through MLS application messages within the group.

### 6.2.1 GroupInfo Replay Mitigation
To mitigate the risk of replay or unauthorized reuse of GroupInfo objects, clients should include expiration timestamps and optionally one-time welcome secrets within the GroupInfo. 

Upon receiving a Welcome and successfully joining the group, the new member should rotate their ephemeral RID. However, RID rotation requires the client to already be in the current epoch, as rotation notifications are delivered via MLS application messages to existing group members. The correct ordering is:

1. Client receives and processes the Welcome, joining the current epoch
2. Client begins receiving messages as a group member
3. Client rotates its RID and notifies peers via MLS application message
4. Peers update their local routing state for the new member

RID rotation must not be attempted before the Welcome is successfully processed. Attempting rotation before joining the epoch means there are no group members to notify, and the rotation notification cannot be delivered.

## 6.3 KeyPackage server ("Keyserver")
To support asynchronous group additions without requiring the target to be online, Runway defines an optional, standalone KeyPackage server. Use of this server is entirely opt-in. Clients that do not upload KeyPackages retain full privacy and fall back to the interactive KeyPackageRequest flow described in 5.6.

### 6.3.1 Design
The KeyPackage server is a simple public store. Clients may upload a pool of KeyPackages in advance, and any peer wishing to add them to an MLS group may fetch one without requiring the target to be online at the time of the add.

The KeyPackage server is independent of the relay server and has no knowledge of RIDs, group membership, or conversation state.

### 6.3.2 Handles
Each client that opts in is assigned a random, opaque handle by the KeyPackage server. This handle is stable for the lifetime of the client's MLS credential and does not rotate like a RID. It carries no user-meaningful information and cannot be enumerated. Clients share their KP handle explicitly and deliberately, typically by including it in their contact token (see 5.6). This is similar to Session's randomized account IDs. The KP handle is not a cryptographic identity. It is purely a lookup key for retrieving KeyPackages.

### 6.3.3 Upload Authentication
To upload KeyPackages, a client must prove ownership of the MLS credential embedded in the KeyPackages it is uploading. The client signs the KeyPackage batch with the private key of its MLS credential. The server verifies this signature before accepting the upload. This prevents an attacker from uploading malformed or other unauthorized KeyPackages under another client's handle.

### 6.3.4 Pool Replenishment
KeyPackages are single-use. When a peer fetches a KeyPackage from the server, that KeyPackage is marked consumed and will not be issued again. Clients are responsible for maintaining a sufficient pool. The KeyPackage server should notify the client when the pool drops below a defined threshold. If the pool is empty, requesters must fall back to the interactive KeyPackageRequest flow. Clients should upload a new batch of KeyPackages upon receiving a low pool notification or upon credential rotation.

### 6.3.5 Contact Token Integration
A contact token (such as a link) can include a KP server handle and domain alongside the initial ephemeral RID. When a KP handle is present, the adder fetches a KeyPackage directly from the KP server, constructs the MLS Welcome immediately, and delivers it to the target's RID without any synchronous interaction. When no KP handle is present, the adder falls back to the interactive KeyPackageRequest flow and must wait for the target to respond before completing the add.

### 6.3.6 Privacy Considerations
Uploading to a KeyPackage server is a deliberate privacy tradeoff. The KP server learns that a handle is active and observes upload and fetch timing. Clients must not upload to any KeyPackage server by default. The opt-in must be an explicit user action. The KP handle and RID are never linked at the protocol level, they only appear together in a contact token the user explicitly generates and shares.

# 7. State Management
## 7.1 Client State
Clients are responsible for managing all state related to MLS groups, including:
- Message history
- Group metadata
- MLS group state

## 7.2 Key Loss
Loss of an MLS credential private key is unrecoverable. Clients that lose their credential MUST generate a new MLS credential and re-establish communication through explicit reintroduction. The protocol does not provide server-assisted recovery mechanisms, as doing so would require persistent secrets or trusted infrastructure. This behavior is consistent with Runway’s capability-based and client-owned identity model.

## 7.3 Epoch Conflicts
MLS is epoch-based. Each commit advances the group to a new epoch, and commits must be applied in order. In a relay model where the server maintains no group state and imposes no ordering on blobs, two members may issue commits referencing the same epoch simultaneously, producing a conflict.

### 7.3.1 Resolution
Runway does not define a server-side ordering mechanism. Clients must handle epoch conflicts locally using the following strategy:

1. The first commit received and applied by a client is accepted as the new epoch.
2. Any conflicting commit referencing the same parent epoch is rejected as stale.
3. The member whose commit was rejected must fetch the latest group state, rebase their changes on top of the new epoch, and attempt to commit again.

This is a last-writer-loses model from the perspective of the discarded commit. Members whose commits are discarded are responsible for detecting the conflict and retrying. Clients should implement a randomized backoff before resubmitting to reduce the likelihood of repeated conflicts in high-activity groups.

## 7.4 Multi-Device
A user may run multiple clients sharing the same MLS credential by exporting and copying their credential private key to additional devices. From the protocol's perspective, all devices sharing a credential are indistinguishable, they present the same cryptographic identity to the group and require no re-adding or additional group membership changes.

### 7.4.1 Credential Export 
Credential export is a manual, user-initiated operation. The client exports the MLS credential private key as an encrypted blob, which the user transfers to the new device through a secure out-of-band channel. The new device imports the credential and generates its own RID. Both devices now share the same MLS identity but have independent routing addresses.

### 7.4.2 Message History
The protocol provides no mechanism for syncing message history to a new device. A device imported with a shared credential receives new messages from the point it comes online but has no access to prior history. Applications may implement encrypted history sync above the protocol layer, but this is explicitly out of scope for the core protocol.

# 8. Metadata considerations
## 8.1 Unavoidable Metadata
The server will inevitably have access to certain metadata, such as:
- IP address
- Timing of messages
- Size of messages
- Recipient IDs (but not group membership or type)
- Delivery fan-out (which can be used to infer group size)

Delivery fan-out is an accepted metadata leak. A message delivered to N recipients reveals that N RIDs were targeted in a single operation, which can be used to infer group size. The protocol does not attempt to hide fan-out count from the server.

Servers also maintain a list of registered RIDs in order to route messages. This list does not contain user-meaningful identifiers.

### 8.1.1 Offline Queue Metadata
Servers that queue blobs for offline recipients inevitably learn the offline status of a given RID. Servers that wish to mitigate this metadata exposure can disable offline queuing entirely.

## 8.2 Mitigation Strategies
Servers must not learn:
- Sender identity
- Group membership
- Conversation type (one-to-one vs group)
- Message content
- Message type

Fan-out count is observable but not mitigated at the protocol level. All other metadata listed above is structurally prevented by the sealed sender construction and opaque blob transport described in sections 3.3 and 5.1.

# 9. Threat model
## 9.1 Adversaries
The protocol is designed to protect against the following adversaries:
- Malicious servers that attempt to learn information about conversations and participants
- Network adversaries that attempt to intercept and analyze messages
- Infrastructure operators that may be compelled to provide access to the server

The protocol does not aim to protect against traffic analysis by a global passive adversary. The protocol does not attempt to hide communication relationships from a malicious peer who is a member of the MLS group either.
## 9.2 Goals
- Confidentiality against server and network attackers
- Minimal metadata exposure
# 10. Federation
Runway supports federated message delivery across independent servers. Each server acts solely as a relay for encrypted blobs and maintains no persistent MLS state.
## 10.1 RID Format
RIDs are expressed in <rid>@server.org format universally. This is the RID format for all Runway deployments, local and federated. <rid> is an ephemeral identifier issued by the recipient's server. server.org specifies the domain of the server hosting the recipient. When sender and recipient share the same server, the domain component is redundant but still present, ensuring all RIDs are globally routable without special casing.

## 10.2 Message Delivery Across Servers
- Clients send MLS ciphertext blobs to <rid>@server.org exactly as they would to a local RID.
- The sending client wraps the MLS ciphertext in a transport envelope containing only the target RID and domain, no other metadata about the sender or group is exposed.
- The federated server verifies the RID and queues the blob for delivery to the local recipient.
- The recipient fetches blobs using their MLS credential to authenticate and decrypt messages.
## 10.3 Security Guarantees
- Federated servers have the same level of security as local servers: they cannot determine sender identity, group membership, or conversation type.
- Forward secrecy and post-compromise security are preserved across servers
- Even a fully malicious federated server cannot decrypt, forge, or replay MLS messages without being detected by the recipient.
Servers can only observe timing, size, and recipient RID, all cryptographic guarantees remain intact.

## 10.4 Initial Contact Across Servers
Initial contact across servers follows the same procedure as local initial contact described in 5.6. RIDs are always expressed in <rid>@server.org format, so contact tokens require no special handling for federation.

# 11. Comparison to Existing Protocols
## 11.1 Matrix
Matrix provides end-to-end encryption via Olm/megOlm, but the server still maintains persistent room state and knows room membership. This is necessary for federation and message history, but exposes metadata about who is in which conversation and when messages are sent. Runway addresses this by making all group state client-owned and opaque to the server, while still supporting secure, one-to-one and group messaging. The server only routes encrypted blobs, never learning group composition or membership changes.
## 11.2 XMPP
XMPP is a flexible, federated messaging protocol. While it can support end-to-end encryption through extensions like OMEMO, it typically relies on the server to manage rooms or relay keys. This can leak metadata about conversations, participants, and message timing. Runway eliminates this server-side knowledge entirely: all conversations are MLS groups managed by clients, and the server only sees opaque blobs and ephemeral routing addresses. This provides stronger privacy and uniform cryptographic guarantees for both one-to-one and group chats.  

## 11.3 SimpleX
SimpleX is the closest existing protocol to Runway in terms of philosophy. It uses no persistent user identifiers, routes messages through relay servers that cannot link senders to recipients, and gives users full control over their contact information. However, SimpleX does not support federation: all relays are independent and there is no standard inter-server delivery protocol. It also predates MLS and uses the Signal-style double ratchet within its own messaging protocol rather than MLS. Runway extends the core ideas of SimpleX with MLS for formally specified group cryptography and a federation model that allows independent servers to interoperate without trusting each other.

# Conclusion
This protocol defines a secure, privacy-respecting messaging system by composing MLS with strict client-side state ownership and minimal server responsibilities. The relay server handles only opaque blob routing via ephemeral RIDs. The optional keyserver provides asynchronous KeyPackage distribution without compromising the relay server's privacy guarantees. Federation is supported natively with identical security properties to local delivery. The protocol achieves strong cryptographic guarantees without relying on trusted infrastructure.
