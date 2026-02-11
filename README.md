# Runway Protocol
## 1. Overview
This document specifies a messaging protocol that provides end to end encrypted communication using the Message Layer Security (MLS) protocol. 

Runway is designed such that:
- Servers only act as relays and coordinators. They do not store state.
- Conversation structure is client owned.
- Servers cannot determine sender identity (via sealed sender), conversation membershsip or conversation type.
- One-to-one and group messaging are indistinguishable to the server.

## 2. Roles and Trust Model
### 2.1 Roles
Clients:
- Must generate and manage all cryptographic keys locally.
- Must maintain MLS group state locally
- Must validate message authenticity and integrity locally
- Must handle message decryption locally

Clients are assumed to be compromised at the endpoint level, MLS provides forward secrecy and post-compromise security to mitigate this risk.
### 2.2 Servers
Servers:
- Treat all messages (or any other info clients may broadcast) as opaque blobs
- Must not maintain group membership or room state
- May queue encrypted blobs if the client is offline; the client can fetch queued blobs when they come online
- May observe unavoidable metadata (timing, size, recipient)
- May use anti-abuse measures (e.g. rate limiting) without compromising privacy guarantees

Servers are assumed to be compromised and malicious. The protocol is designed to minimize the information servers can learn about conversations and participants.

## 3. Cryptography
### 3.1 MLS Usage
- All conversations, including one-to-one chats, are implemented as MLS groups. This provides a consistent security model and feature set across all conversation types.
- One on one conversations use 2 member MLS groups
- Group conversations use MLS groups with 3 or more members
- Clients must process MLS commits and epochs according to the MLS specification
### 3.2 MLS Guarantees
MLS provides:
- End to end encryption: Only group members can decrypt messages
- Forward secrecy: Compromise of current keys does not compromise past messages
- Post-compromise security: After a compromise, future messages can are secure

The protocol relies on MLS for cryptography and does not implement any additional cryptographic primitives. All security guarantees are derived from MLS.

## 4. Message Structure
### 4.1 Overview
The protocol does not define any distinction between:
- One-to-one conversations
- Group conversations
- Channels

All are MLS groups with different numbers of members. The message structure is designed to be opaque to the server, with all relevant information encoded in the MLS ciphertext.
### 4.2 Group Identity
- MLS group IDs must be encrypted inside blobs when transmitted. The server should not be able to determine group membership or type from the group ID.

## 5. Transport Model
### 5.1 Encrypted Blob
The sole unit of transport is the encrypted blob. It contains the reciever ID and the ciphertext. The server relays blobs to the intended recipient(s) without inspecting their contents.

The server must not be able to determine:
- Sender identity (via sealed sender)
- Group membership
- Message type
- Obviously, message content

### 5.2 Delivery
- Clients upload encrypted blobs addressed to recipient IDs. The server relays these blobs to the intended recipients.
- If a recipient is offline, the server may queue the blob for later delivery. The client can fetch queued blobs when they come online.
- Blobs are decrypted locally by the client
### 5.3 Authentication and Integrity
- Recipients authenticate senders using MLS signatures. The server cannot forge messages or impersonate senders due to the use of MLS signatures and sealed sender.
- Sender authenticity must be verified client-side

## 6. Invites and group join
### 6.1 Invites
Group joins are standard MLS. A group member creates a GroupInfo object, which includes the group's context and a public key used to encrypt secrets. The new member downloads the GroupInfo object (it may be encoded in a link or similar) and constructs an MLS commit to request entry. This commit is sent to the group, advancing it to a new epoch. The server relays the commit to all group members, but cannot determine the new member's identity or the fact that a new member joined.
## 7. State Management
### 7.1 Client State
Clients are responsible for managing all state related to MLS groups, including:
- Message history
- Group metadata
- MLS group state

## 8. Metadata considerations
### 8.1 Unavoidable Metadata
The server will inevitably have access to certain metadata, such as:
- Timing of messages
- Size of messages
- Recipient IDs (but not group membership or type)
- Delivery fan-out (which can be used to infer group size)
Servers also have a list of registered users, in order to be able to route messages. 
### 8.2 Mitigation Strategies
Servers must not learn:
- Sender identity
- Group membership
- Conversation type (one-to-one vs group)
- Message content
- Message type
- Group size beyond delivery count (a message sent to 3 recipients could be a one-to-one message sent to 3 different people, or a group message sent to a group of 3)

## 9. Threat model
### 9.1 Adversaries
The protocol is designed to protect against the following adversaries:
- Malicious servers that attempt to learn information about conversations and participants
- Network adversaries that attempt to intercept and analyze messages
- Infrastructure operators that may be compelled to provide access to the server
### 9.2 Goals
- Confidentiality against server and network attackers
- Minimal metadata exposure

# Conclusion
This protocol defines a secure, privacy-preserving messaging system by composing MLS with strict client-side state ownership and minimal server responsibilities. It achieves strong cryptographic guarantees without relying on trusted infrastructure.