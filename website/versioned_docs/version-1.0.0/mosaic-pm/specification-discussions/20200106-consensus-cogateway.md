---
title: Consensus Cogateway
disqus: hackmd
id: version-1.0.0-20200106-consensus-cogateway
original_id: 20200106-consensus-cogateway
---

# Consensus cogateway

| version | Last updated | Component           |
| ------- | ------------ | ------------------- |
| 0.14    | 06/01/2020   | Consensus cogateway |

**Meeting date/time:** N/A
**Editor:** Deepesh Kumar Nath
**Team:** N/A

---

## Overview

The consensus gateway on the origin chain and consensus cogateway on the auxiliary chain establishes a channel for the protocol messages to be communicated between the chains.

The consensus gateway and consensus cogateway use the message bus for atomically communication of cryptographically verifiable messages between two chains.

The following are the type of messages that communicated from the consensus gateway on the origin chain to the consensus cogateway on the auxiliary chain.

- Opening of the kernel.
- Deposit of OST/MOST.
- Activation of an erc20 gateway.

For these messages, the consensus cogateway is permissioned to read the state roots from the origin anchor contract. A merkle patricia proof is provided to proof the declaration of these messages on the consensus gateway in the origin chain.

Consensus cogateway also has permission to mint, increase supply, decrease the supply of the `utMOST` token on the auxiliary chain.

The following are the types of messages that are communicated to consensus gateway on the origin chain from the consensus cogateway on the auxiliary chain.

- Withdrawal of OST/MOST.
- Creation of new erc20 gateway.

## User stories for Gen1-DX/Gen1-BFT

There should be ConsensusCoGateway contract which can facilitate message transfers between origin and auxiliary chains.

1. As a user, it should be possible to confirm open kernel message so that new metablocks can be produced.

2. As a user, it should be possible to prove consensus gateway contract in order to enable storage proof verification.

3. As a user, it should be possible to confirm deposit of MOST, declared on consensus gateway, so that consensus cogateway can mint utMOST for the beneficiary.

4. As a user, I should be rewarded to confirm deposit of MOST (in utMOST, default unwrap).

5. As a user, I should be rewarded to confirm open kernel (Out of scope)

6. As a user, I should be able to withdraw utMOST for MOST through consensus cogateway.

## Tasks

1. Create consensus cogateway contract with below contract as a base contracts.

   1. MessageBus
   2. ConsensusGatewayBase
   3. ERC20GatewayBase
   4. CoConsensusModule
      It should also follow proxy pattern. Create a setup function to initialize the contract storage, MessageOutbox and MessageInbox.

   ```solidity
   contract ConsensusCogateway is MasterCopyNonUpgradable, MessageBus, ConsensusGatewayBase, ERC20GatewayBase, CoConsensusModule
   ```

   ```solidity
       function setup(
       bytes32 _metachainId,
       address _coConsensus,
       ERC20I _utMOST,
       address _consensusGateway,
       uint8 _outboxStorageIndex,
       uint256 _maxStorageRootItems,
       uint256 _metablockHeight
   )
   ```

2. Define confirmOpenKernel method, to confirm already declared kernel in consensusGateway.

   1. It should validate the kernel height i.e new height must be equal to previous height plus 1.
   2. It should increment the nonce of message sender.
   3. It should store kernel hash
   4. It should confirm inbox message by verifying storage proof of consensus gateway using MessageBus.

   ```solidity
   function confirmOpenKernel(
           uint256 _kernelHeight,
           bytes32 _kernelHash,
           uint256 _feeGasPrice,
           uint256 _feeGasLimit,
           address _sender,
           uint256 _blockHeight,
           bytes calldata _rlpParentNodes
       )
   ```

3. Define method `proveConsensusGateway` to verify account proof. It should use MessageInbox for proof verification.

   ```solidity
       function proveConsensusGateway(
           uint256 _blockHeight,
           bytes calldata _rlpAccount,
           bytes calldata _rlpParentNodes
       )
   ```

4. Define method to confirm MOST deposit on consensus gateway contract. It should mint reward for facilitator and remaining UTMOST token for beneficiary.

   Reward is defined as

   ```solidity
       reward = gasPrice * min(gasConsumed, gasLimit)
   ```

   ```solidity
       function confirmDeposit(
           uint256 _amount,
           address payable _beneficiary,
           uint256 _feeGasPrice,
           uint256 _feeGasLimit,
           address _depositor,
           uint256 _blockHeight,
           bytes calldata _rlpParentNodes
       )
           external
           returns (bytes32 messageHash_)
   ```

5. Define position of message outbox and inbox as a constant so that this information could be used to generate and verify proofs for message transfers.

```solidity
    uint8 constant public OUTBOX_OFFSET = uint8(XXX);
    uint8 constant public INBOX_OFFSET = uint8(XXX);
```

## Goals

The Goal for `ConsensusCogateway` contract is to achieve the following on the auxiliary chain.

- The consensus cogateway on the auxiliary chain should establish a unique communication channel with a consensus gateway contract on the origin chain for the communication of the protocol messages. This means that the messages that are declared in the consensus gateway should only be confirmed by the consensus cogateway on an intended auxiliary chain and vice versa.
- In case a new auxiliary chain with the same metachain id is created, when the existing metachain halts, then the consensus cogateway will be able to continue the message communication with the existing consensus gateway on the origin chain.
- The consensus cogateway on the auxiliary chain should be able to confirm the declaration message of open kernel intent in consensus gateway on the origin chain
- The consensus cogateway on the auxiliary chain should be able to confirm the declaration of the MOST/OST deposit intent message in the consensus gateway on the origin chain
- The consensus cogateway on the auxiliary chain should be able to declare withdrawal of OST/MOST intent.
- The consensus cogateway on the auxiliary chain should be able to declare a message for the creation of a new erc20cogateway contract.
- The consensus cogateway on the auxiliary chain should be able to confirm the gateway activation message in the consensus gateway on the origin chain.

## Assumptions

- N/A

## Out of scope

Currently, the following is not in scope. This will be done later

- Declare withdraw of MOST.
- Declare a new cogateway.
- Confirm gateway active.

## Open questions

- N/A

## Approach / Implementation details

- ### Contract architecture

  ```mermaid
      classDiagram

        ConsensusGatewayBase <|-- ConsensusGateway

        ConsensusGatewayBase <|-- ConsensusCogateway

        ERC20GatewayBase <|-- ConsensusGateway

        ERC20GatewayBase <|-- ConsensusCogateway

        MessageBus <|-- ConsensusGateway

        MessageBus <|-- ConsensusCogateway

        ConsensusCogatewayGenesis <|-- ConsensusCogateway

        CoConsensusModule <|-- ConsensusCogateway

        ConsensusModule <|-- ConsensusGateway

      class ConsensusGateway{
            - OUTBOX_OFFSET
            - INBOX_OFFSET
            - setup()
            - declareOpenKernel()
            - deposit()
        }

        class ConsensusCogateway{
            - OUTBOX_OFFSET
            - INBOX_OFFSET
            - setup()
            - proveConsensusGateway()
            - confirmOpenKernel()
            - confirmDeposit()
        }

        class MessageBus{
            - mapping(bytes32 => bool) public inbox
            - mapping(bytes32 => bool) public outbox
            - setupMessageOutbox()
            - setupMessageInbox()
            - declareMessage()
            - confirmMessage()
        }

        class ERC20GatewayBase{
            - ERC20I public token
            - hashDepositIntent()
            - hashWithdrawIntent()
        }

        class ConsensusGatewayBase{
          - ERC20I most;
          - mapping(address => uint256) nonces;
          - uint256 currentMetablockHeight;
          - setup(ERC20I _most,uint256 _currentMetablockHeight)
          - hashOpenKernelIntent()
        }

        class ConsensusCogatewayGenesis{
            - TBD
            - TBD()
        }

        class CoConsensusModule{
          - coConsensus
          - setupCoConsensus(CoConsensusI _coConsensus)
        }

        class ConsensusModule{
          - ConsensusI consensus
          - setupConsensus(ConsensusI _consensus)
        }

  ```

- ### Contracts

  - `ConsensusGatewayBase`
  - `ERC20GatewayBase`
  - `ConsensusModule`
  - `MessageBus`
  - `ConsensusCogatewayGenesis`
  - `ConsensusGateway`
  - `ConsensusCogateway`

- ### Contract flow

#### OUTBOX_OFFSET

The contract should have a public constant variable that indicated the storage index of `MessageBus.outbox` mapping. This will help read the index from the contract while generating the proof data.

Example:

```solidity
uint8 constant public OUTBOX_OFFSET = uint8(index);
```

#### INBOX_OFFSET

The contract should have a public constant variable that indicated the storage index of `MessageBus.inbox` mapping. This will help read the index from the contract while generating the proof data.

Example:

```solidity
uint8 constant public INBOX_OFFSET = uint8(index);
```

#### Setup

Consensus Cogateway follows the proxy pattern. The setup function will be called while deploying the proxy contract for `ConsensusCogateway`.
This can be called only once.
The following will be initialized in this function

- Metachain id
- CoConsensus contract address
- utMOST token address
- ConsensusGateway contract address
- Storage index of outbox mapping in ConsensusGateway contract.
- The maximum number of storage roots to be stored in a circular buffer.
- Initial metablock height.

  The proposed function signature looks like below:

  ```solidity
      function setup(
          bytes32 _metachainId,
          address _coConsensus,
          ERC20I _utMOST,
          address _consensusGateway,
          uint8 _outboxStorageIndex,
          uint256 _maxStorageRootItems,
          uint256 _metablockHeight
      )
          external
  ```

  **Please note: This will be needed for testing purposes only. The initial setup will already be done in the genesis block.**

#### Prove ConsensusGateway contract

This contract will have a function to prove the existence of ConsensusGateway contract address in the origin anchored state root (Merkle Patricia Proof). Once the proof is done this contract will store the storage root of the contract. This function should be called before confirming the open kernel and confirming of deposit message.

Proposed function signature:

```solidity
function proveConsensusGateway(
    uint256 _blockHeight,
    bytes calldata _rlpAccount,
    bytes calldata _rlpParentNodes
)
    external
```

#### Confirm Open Kernel

This contract can confirm the declaration of the open kernel in the origin chain.
This function should keep track of metablock heights. A message can only be confirmed when the kernel height is `+1` of the metablock height.
A Merkle proof should be provided to confirm the declaration of an open kernel in the `ConsensusGateway.outbox` mapping.
This contract should track the confirmed kernel hashes.

Proposed function signature:

```solidity
function confirmOpenKernel(
    uint256 _kernelHeight,
    bytes32 _kernelHash,
    uint256 _feeGasPrice,
    uint256 _feeGasLimit,
    address _sender,
    uint256 _blockHeight,
    bytes calldata _rlpParentNodes
)
    external
    returns (bytes32 messageHash_)
```

#### Confirm Deposit

TBD

---

## Meeting notes

NA

### meeting 2

date: 10/1/2020

Facilitator should be extended with behaviour for ConsensusGateway and ConsensusCogateway.
