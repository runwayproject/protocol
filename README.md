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
## 3.2 MLS Guarantees
MLS provides:
- End to end encryption: Only group members can decrypt messages
- Forward secrecy: Compromise of current keys does not compromise past messages
- Post-compromise security: After a compromise, future messages can remain secure

The protocol relies on MLS for the majority of cryptographic guarantees. The transport layer may use standard public key encryption for sealed sender and routing.
# 4. Message Structure
## 4.1 Overview
The protocol does not define any distinction between:
- One-to-one conversations
- Group conversations
- Channels

All are MLS groups with different numbers of members. The message structure is designed to be opaque to the server, with all relevant information encoded in the MLS ciphertext.    
## 4.2 Group Identity
- MLS group IDs must be encrypted inside blobs when transmitted. The server should not be able to determine group membership or type from the group ID.

# 5. Transport Model
## 5.1 Encrypted Blob
The sole unit of transport is the encrypted blob. It contains the reciever ID and the ciphertext. The server relays blobs to the intended recipient(s) without inspecting their contents.

Recipient IDs are opaque, server-assigned routing identifiers that MAY rotate periodically. The protocol does not assume they are stable or user-meaningful

The server must not be able to determine:
- Sender identity (via sealed sender)
- Group membership
- Message type
- Obviously, message content
### 5.1.1 Blob Structure
Each encrypted blob contains the MLS ciphertext nested inside a sealed sender envelope. While the server sees the recipient's RID and the size of the blob, it cannot decrypt or inspect its contents. This ensures that the sender’s MLS identity remains hidden from the server, even though the RID itself is a routable address. Recipients can authenticate the sender using MLS signatures, independent of the RID.

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

RIDs are merely routing handles the server internally uses, not cryptographic identities. They do not carry any inherent information about the user or their conversations, and their rotation is a key mechanism for maintaining privacy and preventing correlation by the server.

MLS updates, commits, and epochs are independent of RID rotation, clients simply send blobs to the current valid RID of each member.
### 5.5.1 RID Issuance and Rotation
When a client rotates or changes its RID, it must securely notify peers with whom it has established initial contact. This is done through MLS application messages, ensuring that updated RIDs are delivered without revealing routing information to the server or unauthorized parties. Proper RID propagation ensures uninterrupted message delivery while maintaining cryptographic privacy. Rotation of ephemeral RIDs does not require rekeying of MLS groups, only the transport layer address changes. Peers are notified through MLS application messages

## 5.6 Initial Contact and Contact Tokens
Establishing initial communication with a user requires explicit sharing of information. A user must obtain the recipient’s MLS KeyPackage and an initial ephemeral receiver ID through a pre-shared token, link, or other out-of-band mechanism. Possession of this token allows the client to construct a valid MLS commit and begin sending messages, but the server does not learn the identity of the participants or the content of the messages. This ensures that first-contact is a deliberate action controlled by the users themselves and prevents unauthorized clients from initiating conversations or learning about the recipient’s ephemeral routing information. No automatic discovery by username is provided, as this would reveal metadata and compromise privacy.

# 6. Invites and group join
## 6.1 Invites
Group joins are standard MLS. A group member creates a GroupInfo object, which includes the group's context and a public key used to encrypt secrets. The new member downloads the GroupInfo object (it may be encoded in a link or similar) and constructs an MLS commit to request entry. This commit is sent to the group, advancing it to a new epoch. The server cannot determine the new member’s identity, and cannot cryptographically verify group membership changes, though metadata such as message size and fan-out may leak that a group update occurred.

## 6.2 MLS Group Join via Ephemeral Routing
To join an existing MLS group, a new member must receive a GroupInfo object securely, typically through a pre-shared invite token or link that includes the initial ephemeral receiver ID. The new member constructs an MLS join commit and submits it to the group, encrypted under the public keys of current group members. All message delivery, including the initial GroupInfo and the join commit, occurs via ephemeral receiver IDs encapsulated in opaque blobs relayed by the server. The server cannot decrypt group content, determine group membership, or identify the new member; it only observes delivery fan-out and timing. Subsequent updates, including receiver ID rotations, are communicated through MLS application messages within the group.

### 6.2.1 GroupInfo Replay Mitigation
To mitigate the risk of replay or unauthorized reuse of GroupInfo objects, clients SHOULD include expiration timestamps and optionally one-time welcome secrets within the GroupInfo. New members receiving a GroupInfo object SHOULD rotate their ephemeral RIDs immediately upon joining to prevent routing correlation from old invites. These measures help maintain the privacy of group membership and prevent accidental disclosure of the group structure to the server or malicious observers.

# 7. State Management
## 7.1 Client State
Clients are responsible for managing all state related to MLS groups, including:
- Message history
- Group metadata
- MLS group state

## 7.2 Key Loss
Loss of an MLS credential private key is unrecoverable. Clients that lose their credential MUST generate a new MLS credential and re-establish communication through explicit reintroduction. The protocol does not provide server-assisted recovery mechanisms, as doing so would require persistent secrets or trusted infrastructure. This behavior is consistent with Runway’s capability-based and client-owned identity model.

# 8. Metadata considerations
## 8.1 Unavoidable Metadata
The server will inevitably have access to certain metadata, such as:
- IP address
- Timing of messages
- Size of messages
- Recipient IDs (but not group membership or type)
- Delivery fan-out (which can be used to infer group size)
Servers also have a list of registered users, in order to be able to route messages. 

### 8.1.1 Offline Queue Metadata
Servers that queue blobs for offline recipients inevitably learn additional metadata, including the offline status of a given RID/client. Servers that wish to mitigate this metadata exposure can disable it.

## 8.2 Mitigation Strategies
Servers must not learn:
- Sender identity
- Group membership
- Conversation type (one-to-one vs group)
- Message content
- Message type
- Group size beyond delivery count (a message sent to 3 recipients could be a one-to-one message sent to 3 different people, or a group message sent to a group of 3)

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
## 10.1 RID format for Federation
- Receiver IDs are expressed as (human friendly) <rid>@server.org when addressing users on a different server.
- <rid> is an ephemeral identifier issued by the recipient’s server.
- server.org specifies the domain of the server hosting the recipient.

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
Initial contact is the exact same as local initial contact.
# 11. Comparison to Existing Protocols
## 11.1 Matrix
Matrix provides end-to-end encryption via Olm/megOlm, but the server still maintains persistent room state and knows room membership. This is necessary for federation and message history, but exposes metadata about who is in which conversation and when messages are sent. Runway addresses this by making all group state client-owned and opaque to the server, while still supporting secure, one-to-one and group messaging. The server only routes encrypted blobs, never learning group composition or membership changes.
## 11.2 XMPP
XMPP is a flexible, federated messaging protocol. While it can support end-to-end encryption through extensions like OMEMO, it typically relies on the server to manage rooms or relay keys. This can leak metadata about conversations, participants, and message timing. Runway eliminates this server-side knowledge entirely: all conversations are MLS groups managed by clients, and the server only sees opaque blobs and ephemeral routing addresses. This provides stronger privacy and uniform cryptographic guarantees for both one-to-one and group chats.  
# Conclusion
This protocol defines a secure, privacy-preserving messaging system by composing MLS with strict client-side state ownership and minimal server responsibilities. It achieves strong cryptographic guarantees without relying on trusted infrastructure.
