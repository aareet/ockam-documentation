# Threat model

This section models the assumptions and security threats associated to the deployment of a system using Ockam's tools
and services
The guarantees we wish to enforce are described by the [**STRIDE**](https://en.wikipedia.org/wiki/STRIDE_(security))
acronym.
Every threat can be classified in one of six categories:

* Spoofing
* Tampering
* Repudiation
* Information disclosure
* Denial of service
* Elevation of privilege

We list in this document every attack scenario we can think of, that are specific to our architecture or implementation,
and clarify why we are/are not affected and what are the mitigations strategies.

## General assumptions

We make the following assumptions:

- local Ockam nodes are not supposed to be compromised. In particular workers deployed inside a node do not contain
  malicious code
- the Ockam Orchestrator controller is not malicious and performs proper authentication checks
- project nodes started by the Orchestrator controller are not malicious

## Template for an attack scenario

The template for an attack scenario is the following:

| Type         | Description                                                                                             |
|--------------|---------------------------------------------------------------------------------------------------------|
| Kind         | Which type it is, looking at the STRIDE model: spoofing, tampering, etc…                                |
| Assumptions  | What are the basic assumptions, setup, configuration, usages, etc… to start talking about this scenario |
| Threats      | What are we risking? Losing data? Privilege escalation? etc…                                            |
| Mitigations  | What are we doing in order to prevent the threats?                                                      |
| Validation   | How can mitigation are validated/can be validated?                                                      |
| Improvements | Can we do better? how?                                                                                  |
| Status       | Current assumed status: Safe, Can be improved, Unknown, Unsafe                                          |

# Attacks scenarios

## Protocol related

### Credential spam attack

| Type         | Description                                                                                             |
|--------------|---------------------------------------------------------------------------------------------------------|
| Kind         | Denial of service                                                                                       |
| Assumptions  | An attacker can reach the node and send credentials                                                     |
| Threats      | A malicious actor may keep sending valid but useless credentials to fill the target disk and cause DOS. |
| Mitigations  | The threat is very concrete for embedded devices since it could cause the device to effectively brick.  |
| Validation   | None                                                                                                    |
| Improvements | Undefined before defining the wanted behavior                                                           |
| Status       | None without compromising current UX                                                                    |

### Replay attack

| Type         | Description                                                                                                                 |
|--------------|-----------------------------------------------------------------------------------------------------------------------------|
| Kind         | Spoofing                                                                                                                    |
| Assumptions  | Messages are sent through a Secure Channel                                                                                  |
| Threats      | An adversary can replay previously saved messages and pretend to be an authorized sender                                    |
| Mitigations  |                                                                                                                             |
| Validation   | We could record messages sent back and forth and replay them directly from one test                                         |
| Improvements | [Relevant section on Noise Protocol Specification](http://www.noiseprotocol.org/noise.html#out-of-order-transport-messages) |
| Status       | Unsafe                                                                                                                      |

### Forward secrecy - 1

| Type         | Description                                                                                             |
|--------------|---------------------------------------------------------------------------------------------------------|
| Kind         | Information disclosure                                                                                  |
| Assumptions  | Messages are sent through a Secure Channel                                                              |
| Threats      | In case our Identity secret key was compromised, Adversary can decrypt our past Secure Channel messages |
| Mitigations  | Using Noise XX generates ephemeral per-channel encryption material                                      |
| Validation   |                                                                                                         |
| Improvements |                                                                                                         |
| Status       | Safe                                                                                                    |

### Forward secrecy - 2

| Type         | Description                                                                                              |
|--------------|----------------------------------------------------------------------------------------------------------|
| Kind         | Information disclosure                                                                                   |
| Assumptions  | Messages are sent through a Secure Channel                                                               |
| Threats      | In case our Secure Channel was compromised, an adversary can decrypt past messages from the same channel |
| Mitigations  | Using Noise XX generates ephemeral per-channel encryption material                                       |
| Validation   |                                                                                                          |
| Improvements |                                                                                                          |
| Status       | Unsafe                                                                                                   |

### Backward secrecy

| Type         | Description                                                                                                                            |
|--------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Kind         | Information disclosure                                                                                                                 |
| Assumptions  | Messages are sent through a Secure Channel                                                                                             |
| Threats      | In case our Sessions keys and Identity key are compromised, an adversary can decrypt our future messages sent through a Secure Channel |
| Mitigations  | Identity key rotation + Secure Channel rekeying/recreation/ratcheting                                                                  |
| Validation   |                                                                                                                                        |
| Improvements |                                                                                                                                        |
| Status       | Unsafe                                                                                                                                 |

### Rotated Keys on identity remains exploitable by attackers

| Type         | Description                                                                                                                                                                      |
|--------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kind         | Spoofing, Elevation of Privilege                                                                                                                                                 |
| Assumptions  | Entities can rotate their identity keys. The identity identifier remains unchanged after key rotation.                                                                           |
| Threats      | An attacker that has gained possession of some rotated keys of an identity. It is able to impersonate it to nodes that haven’t seen the full, up-to-date chain of that identity. |
| Mitigations  | There are workaround for specific cases, where the full, current identity chain has already been seen by the target                                                              |
| Validation   |                                                                                                                                                                                  |
| Improvements |                                                                                                                                                                                  |
| Status       | Unsafe                                                                                                                                                                           |

A more complete description of the threat can be
found [here](https://app.shortcut.com/ockam-1/story/407/define-how-identity-key-rotation-fit-into-ockam-platform).
Currently an attacker can even present credentials since credentials are tied to identifiers,
and the attacker can get a valid credential for the entity he is impersonating.

### Code related

### Out of memory via many messages

| Type         | Description                                                                                                                                                                                    |
|--------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kind         | Denial of Service                                                                                                                                                                              |
| Assumptions  | One or multiple clients are compromised                                                                                                                                                        |
| Threats      | The whole project node being unusable for long periods of time with relatively small traffic                                                                                                   |
| Mitigations  | Currently, rust queues are bounded, on the elixir side no mitigation is implemented                                                                                                            |
| Validation   | Create a forwarder within a project and then send a few GBs over that forwarder without consuming the content on the receiving end.                                                            |
| Improvements | We could: limit the size of worker queue on the elixir side (with the side effect of breaking during spikes), implement back-pressure or/and rate-limiting for every external incoming traffic |
| Status       | Unsafe [github issue](https://github.com/build-trust/codebase/issues/514)                                                                                                                      |

More details on the threat:

- one or multiple clients could fill the node memory by abusing the queue unbounded size
- the issue can by amplified by routing messages around the target node as much as possible,
  for example by filling the route with repetitive route to the echoer or by sending messages to slower workers

### Deadlock on message deliveries

| Type         | Description                                                                                                                                                                                             |
|--------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kind         | Denial of Service                                                                                                                                                                                       |
| Assumptions  | One or multiple clients are compromised                                                                                                                                                                 |
| Threats      | By filling the router (or any worker) message queue we could trigger a deadlock.                                                                                                                        |
| Mitigations  | When trying to push new messages to a filled queue, the worker would wait, but what if the other worker is also waiting for former worker? Worker A: message -> Worker B  Worker B: message -> Worker A |
| Validation   | We can try to flood a local node repeating the echoer multiple times ([gist](https://gist.github.com/davide-baldo/7dad6147793b627d812748c3165a5c9d))                                                    |
| Improvements | We could timeout when a worker tries to send a message                                                                                                                                                  |
| Status       | Unsafe [github issue](https://github.com/build-trust/codebase/issues/514)                                                                                                                               |

### Man in the middle between nodes

| Type         | Description                                                       |
|--------------|-------------------------------------------------------------------|
| Kind         | Information disclosure                                            |
| Assumptions  | An attacker has access to packets exchanged between nodes         |
| Threats      | Can read private information and/or credentials                   |
| Mitigations  | Implemented Secure Channel in both rust and elixir implementation |
| Validation   | See below                                                         |
| Improvements | None                                                              |
| Status       | Safe                                                              |

Validation:

- the elixir/project side implements encryption through a rust side-car which binds to localhost to avoid any
  interception
- sending a big plain text payload and verifying the entropy level of the content is compatible with encrypted content
- manual verification of the encryption schema and implementation

### Unrestricted messages sent from other nodes

| Type         | Description                                                                         |
|--------------|-------------------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                                 |
| Assumptions  | Another node has access to a node message bus after connecting via a secure channel |
| Threats      | Reach any worker using the node router                                              |
| Mitigations  | Limit Outgoing Access Control for messages originated from other nodes              |
| Validation   | Tests for individual components and design & code review                            |
| Improvements |                                                                                     |
| Status       | Unsafe                                                                              |

### Unrestricted messages from outgoing TCP connections - 1

| Type         | Description                                                                                                                                                                                                 |
|--------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                                                                                                                                                         |
| Assumptions  | TCP connection is created during multiaddr resolution when tcp connection is followed by a secure channel (e.g. /node/name/service/api or /project/name/service/api or /ip4/127.0.0.1/tcp/4000/service/api) |
| Threats      | Reach any worker on the node message bus                                                                                                                                                                    |
| Mitigations  | Limit TCP Connections to be able to forward messages to the corresponding Secure Channels decryptor only                                                                                                    |
| Validation   | Set of tests in rust library                                                                                                                                                                                |
| Improvements |                                                                                                                                                                                                             |
| Status       | Safe                                                                                                                                                                                                        |

### Unrestricted messages from outgoing TCP connections - 2

| Type         | Description                                                                |
|--------------|----------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                        |
| Assumptions  | TCP connections are created via library API properly using sessions        |
| Threats      | Reach any worker using the node router                                     |
| Mitigations  | Setup Sessions so that outgoing messages can only reach intended addresses |
| Validation   | Set of tests in rust library                                               |
| Improvements |                                                                            |
| Status       | Safe                                                                       |

### Unrestricted messages from outgoing TCP connections - 3

| Type         | Description                                                                                                                                             |
|--------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                                                                                                     |
| Assumptions  | TCP connections are created manually via ockam command (e.g. `tcp-connection create`)                                                                   |
| Threats      | Reach any worker using the node router                                                                                                                  |
| Mitigations  | Either generate session_id for such connections and figure out how to propagate that id to the consumers of these connections, or disable such commands |
| Validation   |                                                                                                                                                         |
| Improvements |                                                                                                                                                         |
| Status       | Unsafe                                                                                                                                                  |

### Unrestricted messages from outgoing TCP connections - 4

| Type         | Description                                                  |
|--------------|--------------------------------------------------------------|
| Kind         | Privilege elevation                                          |
| Assumptions  | TCP connections are created via Library API without Sessions |
| Threats      | Reach any worker using the node router                       |
| Mitigations  | Do not allow creating such TCP connections                   |
| Validation   |                                                              |
| Improvements |                                                              |
| Status       | Unsafe                                                       |

### Unrestricted messages from outgoing TCP connections - 5

| Type         | Description                                                                                                                                            |
|--------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                                                                                                    |
| Assumptions  | A TCP listener is created by default for a node created via ockam command                                                                              |
| Threats      | Reach any worker using the node router                                                                                                                 |
| Mitigations  | Use Sessions to restrict such listener to access only NodeManager, Secure Channel Listener (and optionally workers that are required for our examples) |
| Validation   |                                                                                                                                                        |
| Improvements |                                                                                                                                                        |
| Status       | Unsafe                                                                                                                                                 |

### Unrestricted messages from secure channel - 1

| Type         | Description                                                                             |
|--------------|-----------------------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                                     |
| Assumptions  | Secure Channel is created during multiaddr resolution                                   |
| Threats      | Reach any worker using the node router                                                  |
| Mitigations  | Limit Secure Channel to be able to forward messages to the corresponding consumers only |
| Validation   |                                                                                         |
| Improvements |                                                                                         |
| Status       | Unsafe                                                                                  |

### Unrestricted messages from secure channel - 2

| Type         | Description                                                                   |
|--------------|-------------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                           |
| Assumptions  | Secure Channels/Listeners are created via Library API properly using Sessions |
| Threats      | Reach any worker using the node router                                        |
| Mitigations  | Setup Sessions so that outgoing messages can only reach intended Addresses    |
| Validation   | Set of tests in rust library                                                  |
| Improvements |                                                                               |
| Status       | Safe                                                                          |

### Unrestricted messages from secure channel - 3

| Type         | Description                                                                                                                                                    |
|--------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                                                                                                            |
| Assumptions  | Secure Channels/Listeners are are created manually via ockam command (e.g. secure-channel create or secure-channel-listener create)                            |
| Threats      | Reach any worker using the node router                                                                                                                         |
| Mitigations  | Either generate session_id for such channels/listeners and figure out how to propagate that id to the consumers of these connections, or disable such commands |
| Validation   |                                                                                                                                                                |
| Improvements |                                                                                                                                                                |
| Status       | Unsafe                                                                                                                                                         |

### Unrestricted messages from secure channel - 4

| Type         | Description                                                            |
|--------------|------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                    |
| Assumptions  | Secure Channels/Listeners are created via Library API without Sessions |
| Threats      | Reach any worker using the node router                                 |
| Mitigations  | Do not allow creating such Secure Channels/Listeners connections       |
| Validation   |                                                                        |
| Improvements |                                                                        |
| Status       | Unsafe                                                                 |

### Unrestricted messages from secure channel - 5

| Type         | Description                                                                        |
|--------------|------------------------------------------------------------------------------------|
| Kind         | Privilege elevation                                                                |
| Assumptions  | Secure Channel Listener is created by default for a node created via ockam command |
| Threats      | Reach any worker using the node router                                             |
| Mitigations  | Use Sessions to restrict such listener to access only intended Addresses           |
| Validation   |                                                                                    |
| Improvements |                                                                                    |
| Status       | Unsafe                                                                             |

### Node manager can be reached in an authority node

| Type         | Description                                                                                                       |
|--------------|-------------------------------------------------------------------------------------------------------------------|
| Kind         | Privilege Escalation                                                                                              |
| Assumptions  | An identity is authenticated and can reach the authority node.                                                    |
| Threats      | Can send requests to NodeManager to spawn resources, a skilled attacker could probably use it as an attack vector |
| Mitigations  | The NodeManager is not started when starting an authority node                                                    |
| Validation   | Automated test trying to send a request to the node manager                                                       |
| Improvements | Add ABAC rules validation for every operation                                                                     |
| Status       | Safe                                                                                                              |

### Time-based attack on credential validation

| Type         | Description                                                                                                                                                                             |
|--------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Kind         | Information Disclosure                                                                                                                                                                  |
| Assumptions  | The attacker has access to the node, requiring credentials                                                                                                                              |
| Threats      | The credential itself could be leaked by sending thousands of credential authentications, and by measuring latency, the secret value can be inferred if the secret is validated naively |
| Mitigations  | Credentials are verified in a fixed time by comparing hashes                                                                                                                            |
| Validation   | Manually validate the timing never changes, regardless of the size of the internal credential                                                                                           |
| Improvements | None                                                                                                                                                                                    |
| Status       | Safe                                                                                                                                                                                    |

### Rust NIST P-256 Implementation has not been audited

| Type         | Description                                                                                                             |
|--------------|-------------------------------------------------------------------------------------------------------------------------|
| Kind         | Spoofing                                                                                                                |
| Assumptions  | An attacker can read the encrypted traffic between 2 nodes                                                              |
| Threats      | An attacker may exploit a vulnerability within the library to break the encryption, leak secrets, or tamper the content |
| Mitigations  | Audit the implementation                                                                                                |
| Validation   | None                                                                                                                    |
| Improvements | None                                                                                                                    |
| Status       | Unknown (we validated that rust v1.67 generates fixed-time instructions for amd64/x86-64)                               |

### Infrastructure related

### Physical disks of the Orchestrator could be compromised

| Type         | Description                                                                  |
|--------------|------------------------------------------------------------------------------|
| Kind         | Information disclosure                                                       |
| Assumptions  | Old AWS disks are replaced/thrown away or sold, or AWS disks get stolen      |
| Threats      | We are risking leaking credentials and secrets with vault                    |
| Mitigations  | None                                                                         |
| Validation   | Since we cannot have physical access, we don’t think there is a way          |
| Improvements | Disk encryption within EBS, where the secrets and/or credentials are stored. |
| Status       | Unsafe                                                                       |

### Compromised Cargo dependency

| Type         | Description                                                                                          |
|--------------|------------------------------------------------------------------------------------------------------|
| Kind         | Elevation of privilege                                                                               |
| Assumptions  | One of 387+ packages we are dependent on, gets compromised.                                          |
| Threats      | Malicious code could be injected, granting complete access to every device/installation              |
| Mitigations  | GitHub's dependency bot notifies when such an incident is publicly known                             |
| Validation   | None possible                                                                                        |
| Improvements | Aggressively reduce dependencies. Subscribe to paid security services to get a quicker notification. |
| Status       | Unknown                                                                                              |
