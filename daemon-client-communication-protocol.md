
[![hackmd-github-sync-badge](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ/badge)](https://hackmd.io/pym9JPVlRK-RfQGOUv26aQ)

<pre>
  State: draft
  Created: 2021-02-03
</pre>

# Instructions & State Digests

## Overview

This RFC specifies the messages exchanged between a user's client and their own daemon.

```
  sk,pk       instructions     pk
 -----------  ------------> -------------
 | client  |                | daemon    |
 -----------  <-----------  -------------
              state digests
```
As sketched in the above graphic, the `client`→`daemon` route consists of instructions from the client to daemon that lead to a state transition of an ongoing swap, and the `daemon`→`client` route consists of state digests sent to the client whenever a new choice becomes available to the client. Both instructions and state digests follow the [TLV format specified in the Lightning RFCs](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#type-length-value-format).

### Security considerations
> [name=Lederstrumpf][color=violet] Possibly, this has been written elsewhere already, or would find a better home in another RFC

From a security perspective, an important distinction between the client and the daemon is that the daemon only knows public keys - private keys are the privy treasure of the client. Nonetheless, the daemon MUST be viewed as a trusted component, since it exclusively verifies the correctness of the counterparty's data, controls the swap state, and can misreport progression of the swap to the client whenever the progress does not require a signature from the client. For instance, if the client is Bob who initially owns BTC in a swap, and the refund path is invoked, if the client signs the `BTX_spend` transaction and instructs the daemon to relay it, a malicious daemon could abstain from relaying it, resulting in a loss of funds for Bob, if he does not detect this error and submit the signed transaction via an alternate route before Alice can submit the `BTX_claim` transaction to punish Bob.

## Table of Contents

[TOC]

## Instructions

There are three types of instructions:
1. Cryptographic material: relevant during the initialization phase
   - signatures (partial and finalized)
   - keys (public keys and Monero private view keys)
   - MuSig2 protocol messages
   - Zero knowledge proofs of DLEQ across groups
2. Transaction messages following [PSBT standard](https://github.com/bitcoin/bitcoin/blob/master/doc/psbt.md) & [BIP 174](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki) : relevant during all swap steps
3. Control flow messages
   - accept a step in the swap process
   - swap cancel by the user
   
### Cryptographic material
#### 1. Initialization step
These are the messages that the client can send to the daemon in context of the initialization step.
##### `send_initialization_parameters_{alice,bob}`
Provides the counter-party daemon with all the information required for the initialization step of a swap.
###### Type
X `(init)`
###### Data
- `B_{a,b}`: `base58`: the destination address for a success swap or the address used for initally locking the funds respectively.
- `B_{a,b}_r`: `base58`: Bitcoin public key used for the recovery path
- `k_{a,b}_v`: `edward25519` scalar: private view key share, needed by the counterparty daemon for calculating the full private view key $K^v$
- `K_{a,b}_s`: `edward25519` curve point: public spend key, needed by the counterparty daemon for calculating the full public spend key $K^s$ and verifying the DLEQ proof
- `dleq_proof_{a,b}`: `DLEQ proof`: DLEQ proof for the equal discrete logarithm of the client's Bitcoin key $B_i$ and their Monero public spend key share $K^s_i$

###### TLV Data
- `[u16: btc_addr_len]`
- `[btc_addr_len * byte: B_{a,b}]`
- `[u16: btc_addr_len]`
- `[btc_addr_len * byte: B_{a,b}_r]`
- `[u16: ed25519_scalar_len]`
- `[ed25519_scalar_len * byte: k_{a,b}_v]`
- `[u16: ed25519_point_len]`
- `[ed25519_point_len * byte: K_{a,b}_v]`
- `[u16: dleq_proof_len]`
- `[dleq_proof_len * byte: dleq_proof_{a,b}]`

#### 2. Bitcoin Locking Step
##### `send_signed_btx_refund_bob`
Only called by Bob's daemon: provides Bob's daemon with a signature on the unsigned `BTX_refund` transaction provided by the daemon via (`TODO`: link applicable state digest)

###### Type
X + 1 (`sig_btx_refund_bob`)
`X`
###### 
- `sig_btx_refund_bob`: `ECDSA signature`
##### `send_signed_btx_spend_alice`
Only called by Alice's daemon: provides Alice's daemon with a signature on the unsigned `BTX_spend` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
X + 2 (`sig_btx_spend_alice`)
###### Data
- `sig_btx_spend_alice`: `ECDSA signature`
###### TLV Data
- `[u16: sig_btx_len]`
- `[sig_btx_len * type: sig_btx_spend_alice]`
`sig_btx_spend_alice`: `ECDSA signat
Only called by Alice's daemon: provides Alice's daemon with a signature on the unsigned `BTX_refund` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
X + 3 `(sig_btx_refund_alice)`
###### Data
- `sig_btx_refund_alice`: `ECDSA signature`
###### TLV Data
- `[u16: sig_btx_len]`
- `[sig_btx_len * type: sig_btx_refund_alice]`
#### 3. Monero Locking Step
##### `send_signed_btx_buy_bob`
Only called by Bob's daemon: provides Bob's daemon with a signature on the unsigned `BTX_buy` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
X + 4 `(sig_btx_buy_bob)`
`void | boo
- `sig_btx_buy_bob`: `ECDSA signature`
###### TLV Data
- `[u16: sig_btx_len]`
- `[sig_btx_len * type: sig_btx_buy_bob]`
##### `send_signed_xtx_lock_alice`
Only called by Alice's daemon: provides Alice's daemon with a signature on the unsigned `XTX_lock` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
X + 5 `(sig_xtx_lock_alice)`
###### Data
- `sig_xtx_lock_alice`: `EdDSA signature`
###### TLV Data
- `[u16: sig_xtx_len]`
- `[sig_xtx_len * type: sig_xtx_lock_alice]`

#### 4. Swap Step
##### `send_signed_btx_buy_alice`
Only called by Alice's daemon: provides Alice's daemon with a signature on the unsigned `BTX_buy` transaction provided by the daemon via (`TODO`: link applicable state digest)
###### Type
X + 6 `sig_btx_buy_alice`
###### Data
- `sig_btx_buy_alice`: `ECDSA signature`
###### TLV Data
- `[u16: sig_btx_len]`
- `[sig_btx_len * type: sig_btx_buy_alice]`

## State digests
State digests are messages sent by the daemon to the client to relay all information needed to construct the client interface and update the swap state. These messages both allow the client to display the correct actions a user can perform and create the necessary instructions consumed daemon to continue the swap protocol.
### Format (ROUGH SKETCH)
- current marking of Petri net representation of swap (i.e. vector of token counts per slot)?
- and/or transitions available for firing (superfluous if we send the current marking, but that route requires more data transfer & calculation on the client's part )
> [name=Lederstrumpf][color=violet]
> I would prefer sending markings, in lieu of baking in the transitions as a stripped down representation - my expectation is that while requiring more functionality on the client's part, it will be easier to keep in sync with daemon development down the line, and safer to reason about.
- data required for signatures (data required for firing a given transition)

### `send_state_digest`
Provides the client with the current state digest. By applying the fired transitions to the current petri net of the client, the client can infer which transitions are available to fire.
#### Type
Y `(state_digest)`
#### Data
- `fired_transitions`: `[u16]` (vector of # of firings per transition since last state digest)
- `marking_hash`: `SHA256` (hash of the current marking so that client can verify it applied the transitions correctly)
- `fireable_transition_data`: `{transition index -> required data}` (map of fireable transitions to the data that the client requires to produce the authentication the daemon requires to fire it, such as transaction signatures)
###### TLV Data
- `[u16: ft_len]`
- `[ft_len * type: fired_transitions]`
- `[SHA256: marking_hash]`
- `[u16: ftd_len]`
- `[ftd_len * type: fireable_transition_data]`