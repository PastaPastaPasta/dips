# Dash Improvement Proposal (DIP): Secure 'Onion' Routing Protocol for Masternodes
### Lasagna Layer Privacy Protocol

## Abstract

This DIP introduces a refined secure onion (or lasagna) routing protocol for the Dash network, aiming to significantly 
enhance privacy and security in transaction and data communication among masternodes and clients. This proposal builds on the 
existing quorum-based structure of Dash's masternode network to provide a robust, decentralized alternative to 
traditional anonymity networks, prioritizing optimized performance, user privacy, and scalability.

## 1. Introduction

### 1.1 Background
Building upon Dash's masternode functionalities, this protocol is designed to fortify the privacy aspects, particularly
focusing on concealing the origins of transactions and data packets within the network. It would be recommended that
all users broadcast their transaction using this protocol, as doing so will make it significantly harder to link a user's
IP address to their specific transaction. This protocol would also provide significant value for users using integrated
CoinJoin as their IP address and inputs would no longer be linkable via the masternode. This un-linkability would also
enable the implementation of blinding in the coinjoin system which would remove the observability of input output
correlation by a masternode; this should be covered in greater depth in another document.

This protocol should not be used for messages between masternodes, as this protocol significantly increases latency and
computation for messages.

Additionally, this protocol still remains valuable after an implementation of P2P encryption as this system removes the
ability for any single node to correlate a message to an IP address.

### 1.2 Rationale
Integrating a native onion routing mechanism within Dash's infrastructure offers streamlined performance and minimizes
dependencies on external systems, thereby enhancing the overall user experience, especially for light clients.

## 2. Protocol Overview

### 2.1 Masternode Identification
Masternodes continue to be identified through their unique ProTx hash. Considerations for shorter representations
of this hash should be meticulously assessed to ensure they do not compromise the network's integrity or the validation
processes of full nodes.

### 2.2 Data Structure and Standardization
The protocol adapts to the variable sizes of messages. Each message, irrespective of its initial size, is padded to the
nearest multiple of 1024 bytes (1 kilobyte). This standardization in message size is crucial for preventing size-based
analysis attacks and ensuring uniformity in data packets, thereby enhancing anonymity. It should be noted in this design,
that only the initial fully wrapped encrypted message is padded up to 1KB, as such after the first decryption, the size
may be decreased.

#### 2.2.1 Consideration for Packet Size
In optimizing the protocol for real-world applications, the optimal size of a packet should be considered. It is
possible we should pad messages to a greater padded to size if that results in no or minimal downsides as it would
result in a better privacy set.

### 2.3 Layered Encryption and Routing
Messages undergo multiple layers of encryption, with each layer designated for a specific masternode. The routing
mechanism ensures that each masternode decrypts its respective layer, identifies the next recipient from the encrypted
packet, and forwards the message accordingly. This process repeats until the final layer is decrypted by the intended
masternode, which then broadcasts the original message to the network.

An encrypted message will look like `{encrypted_blob}`. If a message fails decryption, the sending peer
should be marked misbehaving. Upon decryption, the message will then be either
`{recipient_protx_hash, encrypted_blob}` or `{all_zero_protx_hash, unencrypted_message}`. In the case of an unencrypted message, it should be
submitted to the local inventory system and processed like any other message. By using an all zero ProTx hash to signal
completion, it is obvious whether a message needs to be forwarded, or has reached its final destination. 

#### 2.3.1 Quorum-based Optimization
The protocol leverages the interconnected nature of quorums to optimize routing paths, potentially accelerating the
message delivery process by minimizing hops and latency, and thereby enhancing the speed and efficiency of the network.

## 3. Security Considerations

### 3.1 Anonymity and Standardization
By standardizing the message sizes and employing a sophisticated encryption mechanism, the protocol ensures the
anonymity of the senders, effectively masking transaction origins and protecting against traffic analysis.

### 3.2 Encryption and Security
The encryption methodology must be rigorously selected for its robustness and resilience. The choice of algorithms,
key lengths, and schemes should be driven by a comprehensive security-first approach, balancing performance and
impenetrability. Obviously, the presence of BLS keys already in the DML means we should investigate using these keys
for the encryption. However, this may not be possible, or efficient enough for this purpose. In such a case, other
primitives should be evaluated. 

In the result that we determine a different primitive should be used, it would likely result in requiring masternodes
to re-register in the DML to add this new public key.

## 4. Network Impact and Scalability

### 4.1 Handling Increased Load
The implications of the additional data load and processing requirements on masternodes necessitate a thorough analysis.
The incentive mechanisms may require recalibration to adequately compensate for the increased operational load.

### 4.2 Ensuring Reliability and Fault Tolerance
The system must maintain high reliability and resilience, ensuring efficient message delivery and robustness against
node failures or adversarial actions.

## 5. Comparative Analysis with Tor

### 5.1 Advantages of the Native System
- **Seamless Integration**: Fully integrated within the Dash ecosystem, negating the need for additional setups or installations.
- **Network Optimizations**: Utilizes quorum dynamics for faster, more efficient routing.
- **User-Centric Design**: Provides a straightforward, intuitive solution for light clients, fostering wider adoption and usage.

### 5.2 Limitations and Considerations
- **Absence of Onion Services**: Unlike Tor, this protocol does not offer services for node or server location hiding.
- **Dependence on Network Infrastructure**: The efficacy and security of the protocol are inherently tied to the robustness of Dash's masternode system.

## 6. Future Work and Enhancements

- **In-depth Analysis of Packet Size**: Detailed exploration of aligning message sizes with standard TCP packet sizes, considering encryption overhead and network efficiency.
- **Refinement of Encryption Strategies**: Selection and optimization of encryption algorithms and key management protocols, with a focus on maximizing security while maintaining system performance.
- **Incentive Structure Review**: Comprehensive analysis and potential restructuring of incentives for masternode operators, taking into account the augmented workload and operational demands.
- **Extensive Protocol Testing**: Rigorous testing regimes to assess and enhance the protocol's
