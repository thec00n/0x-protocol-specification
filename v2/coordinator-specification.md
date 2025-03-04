# 0x 2.0.0 Coordinator Specification

1.  [Architecture](#architecture)
1.  [Message Types](#message-types)
    1. [Orders](#orders)
    1. [Transactions](#transactions)
    1. [Approvals](#approvals)
1.  [Contracts](#contracts)
    1.  [Coordinator](#coordinator)
    1.  [CoordinatorRegistry](#coordinatorregistry)
1.  [Signature Types](#signature-types)
1.  [Contract Events](#events)
1.  [Reference Coordinator Server](#reference-coordinator-server)
1.  [Standard Coordinator API](#standard-coordinator-api)

# Architecture

[0x version 2.0](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md) introduced the concept of a [0x transactions](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#transactions), a new way to interact with the [`Exchange` contract](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#exchange). 0x transactions allow external accounts or contracts to execute Exchange methods in the context of the transaction signer, which makes it possible to add custom logic to the execution of trades or cancels.

The coordinator model makes heavy use of this concept in order to add an additional layer of verification to `Exchange` contract interactions. An order may specify a single coordinator's address that is responsible for approving any fills of the order. The coordinator can set custom rules that may be used to prevent trade collisions, enable free and instant cancels, or add time delays to filling order.

A coordinator has 2 components that differentiate it from a traditional relayer:

1. The [`Coordinator` contract](#coordinator), which is a [0x extension contract](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#filter-contracts) that verifies transactions have been approved by the correct set of coordinators.
1. A [coordinator server](#reference-coordinator-server) that approves or rejects 0x transactions under different conditions.

This specification will describe a specific implementation of each component that intends to create a market structure that is more favorable for liquidity providers while allowing for liquidity to be consumed by smart contracts. See the reference server [design choices](#design-choices) section for more information. The flow for filling an order with the coordinator model is as follows:

1. A taker selects the desired order(s) and creates a valid signed 0x transaction to fill the order(s) (e.g. `batchFillOrdersNoThrow`).
2. The taker looks up the Coordinator server endpoint(s) corresponding to the order(s) using the [Coordinator Registry](#CoordinatorRegistry) smart contract.
3. The taker sends a fill request with the signed 0x transaction to the coordinator server that corresponds to each order.
4. After a 1 second delay, the coordinator server responds with a signed approval.
5. The taker submits the signed 0x transaction and approval to the `Coordinator` extension contract by calling `Coordinator.executeTransaction`.
6. The `Coordinator` contract verifies that the approval is valid/unexpired and executes the 0x transaction by calling `Exchange.executeTransaction`.

<div style="text-align: center;">
<img src="./img/0x_v2_coordinator.png" style="padding-bottom: 20px; padding-top: 20px;" width="80%" />
</div>

# Message Types

## Orders

The [`Coordinator` contract](#coordinator) is compatible with orders where the [`senderAddress`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#senderaddress) is equal to the address of this contract or the null address. If an order's `senderAddress` is null, that order may be filled or cancelled through the `Coordinator` contract or directly through the `Exchange` contract. For a full specification of the order schema, please see the [orders section](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#orders) of the 0x 2.0 specification.

For the purposes of this specification, orders that specify the `Coordinator` contract as the `senderAddress` will be referred to as "coordinator orders".

## Transactions

The `Coordinator` contract processes regular [0x transaction](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#transactions) messages, using the `Exchange` 2.0 EIP712 domain header (see the Solidity type [here](#zeroextransaction)). Transactions may be signed with any valid [0x 2.0 signature type](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#signature-types).

In Solidity, this message is represented as:

```
struct ZeroExTransaction {
    uint256 salt;           // Arbitrary number to ensure uniqueness of transaction hash.
    address signerAddress;  // Address of transaction signer.
    bytes data;             // AbiV2 encoded calldata.
}
```

## Approvals

0x transactions that call any `Exchange` fill methods must be approved by a coordinator. The approval includes the following fields (see the Solidity type [here](#coordinatorapproval)):

| Parameter                     | Type    | Description                                                                           |
| ----------------------------- | ------- | ------------------------------------------------------------------------------------- |
| txOrigin                      | address | Address of Ethereum transaction signer that is allowed to execute this 0x transaction |
| transactionHash               | bytes32 | EIP712 hash of the 0x transaction.                                                    |
| transactionSignature          | bytes   | Signature of 0x transaction.                                                          |
| approvalExpirationTimeSeconds | uint256 | Timestamp in seconds for which the approval expires.                                  |

In Solidity, this message is represented as:

```
struct CoordinatorApproval {
    address txOrigin;                       // Required signer of Ethereum transaction that is submitting approval.
    bytes32 transactionHash;                // EIP712 hash of the transaction, using the `Exchange` contract's domain separator.
    bytes transactionSignature;             // Signature of the 0x transaction.
    uint256 approvalExpirationTimeSeconds;  // Timestamp in seconds for which the approval expires.
}
```

The hash of an approval can be calculated with:

```
// EIP191 header for EIP712 prefix
string constant internal EIP191_HEADER = "\x19\x01";

// EIP712 Domain Name value for the Coordinator
string constant internal EIP712_COORDINATOR_DOMAIN_NAME = "0x Protocol Coordinator";

// EIP712 Domain Version value for the Coordinator
string constant internal EIP712_COORDINATOR_DOMAIN_VERSION = "1.0.0";

// Hash of the EIP712 Domain Separator Schema
bytes32 constant internal EIP712_DOMAIN_SEPARATOR_SCHEMA_HASH = keccak256(abi.encodePacked(
    "EIP712Domain(",
    "string name,",
    "string version,",
    "address verifyingContract",
    ")"
));

// Hash for the EIP712 Coordinator approval message
bytes32 constant internal EIP712_COORDINATOR_APPROVAL_SCHEMA_HASH = keccak256(abi.encodePacked(
    "CoordinatorApproval(",
    "address txOrigin,",
    "bytes32 transactionHash,",
    "bytes transactionSignature,",
    "uint256 approvalExpirationTimeSeconds",
    ")"
));

bytes32 EIP712_COORDINATOR_DOMAIN_HASH = keccak256(abi.encodePacked(
    EIP712_DOMAIN_SEPARATOR_SCHEMA_HASH,
    keccak256(bytes(EIP712_COORDINATOR_DOMAIN_NAME)),
    keccak256(bytes(EIP712_COORDINATOR_DOMAIN_VERSION)),
    uint256(address(this))
));

bytes32 hashStruct = keccak256(abi.encodePacked(
    EIP712_COORDINATOR_APPROVAL_SCHEMA_HASH,
    approval.txOrigin,
    approval.transactionHash,
    keccak256(approval.transactionSignature)
    approval.approvalExpirationTimeSeconds,
));

bytes32 approvalHash = keccak256(abi.encodePacked(
    EIP191_HEADER,
    EIP712_COORDINATOR_DOMAIN_HASH,
    hashStruct
));

```

The hash of an approval must be signed by a coordinator in order for the approval to be valid. See the [signature types](#signature-types) section for more information on creating valid signatures.

# Contracts

## Coordinator

The `Coordinator` contract is the single entry point for executing transactions relevant to coordinator orders.

### executeTransaction

The `Coordinator` contract contains a single function that is allowed to execute approved transactions and alter blockchain state.

When interacting with coordinator orders, the following Exchange methods must be called through `executeTransaction` on the `Coordinator` contract:

| Method                                                                                                                                         | Approval required |
| ---------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| [`fillOrder`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#fillorder)                             | Yes               |
| [`fillOrKillOrder`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#fillorkillorder)                 | Yes               |
| [`fillOrderNoThrow`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#fillordernothrow)               | Yes               |
| [`batchFillOrders`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#batchfillorders)                 | Yes               |
| [`batchFillOrKillOrders`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#batchfillorkillorders)     | Yes               |
| [`batchFillOrdersNoThrow`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#batchfillordersnothrow)   | Yes               |
| [`marketBuyOrders`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#marketbuyorders)                 | Yes               |
| [`marketBuyOrdersNoThrow`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#marketbuyordersnothrow)   | Yes               |
| [`marketSellOrders`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#marketsellorders)               | Yes               |
| [`marketSellOrdersNoThrow`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#marketsellordersnothrow) | Yes               |
| [`matchOrders`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#matchorders)                         | Yes               |
| [`cancelOrder`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#cancelorder)                         | No                |
| [`batchCancelOrders`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#batchcancelorders)             | No                |
| [`cancelOrdersUpTo`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#cancelordersupto)               | No                |

`executeTransaction` will revert under the following conditions:

- The `tx.origin` (Ethereum transaction signer) differs from the passed in `txOrigin` parameter.
- Any of the coordinator orders include a `feeRecipientAddress` that do not have a corresponding valid signature in `approvalSignatures`.
- Any of the values in `approvalExpirationTimeSeconds` are less than or equal to the timestamp of the block in which this transaction is mined.
- Each item in `approvalSignatures` does not have a corresponding item in `approvalExpirationTimeSeconds`.
- The `Exchange` function call in `transaction.data` reverts for any reason.

```
/// @dev Executes a 0x transaction that has been signed by the feeRecipients that correspond to each order in the transaction's Exchange calldata.
/// @param transaction 0x transaction containing salt, signerAddress, and data.
/// @param txOrigin Required signer of Ethereum transaction calling this function.
/// @param transactionSignature Proof that the transaction has been signed by the signer.
/// @param approvalExpirationTimeSeconds Array of expiration times in seconds for which each corresponding approval signature expires.
/// @param approvalSignatures Array of signatures that correspond to the feeRecipients of each order in the transaction's Exchange calldata.
function executeTransaction(
    LibZeroExTransaction.ZeroExTransaction memory transaction,
    address txOrigin,
    bytes memory transactionSignature,
    uint256[] memory approvalExpirationTimeSeconds,
    bytes[] memory approvalSignatures
)
    public;
```

### assertValidCoordinatorApprovals

`assertValidCoordinatorApprovals` is a read-only helper function used for validating transaction approvals. Note that this function cannot detect failures that would occur when the 0x transaction is being executed by the `Exchange` contract.

```
/// @dev Validates that the 0x transaction has been approved by all of the feeRecipients
///      that correspond to each order in the transaction's Exchange calldata.
/// @param transaction 0x transaction containing salt, signerAddress, and data.
/// @param txOrigin Required signer of Ethereum transaction calling this function.
/// @param transactionSignature Proof that the transaction has been signed by the signer.
/// @param approvalExpirationTimeSeconds Array of expiration times in seconds for which each corresponding approval signature expires.
/// @param approvalSignatures Array of signatures that correspond to the feeRecipients of each order in the transaction's Exchange calldata.
function assertValidCoordinatorApprovals(
    LibZeroExTransaction.ZeroExTransaction memory transaction,
    address txOrigin,
    bytes memory transactionSignature,
    uint256[] memory approvalExpirationTimeSeconds,
    bytes[] memory approvalSignatures
)
    public
    view;
```

### getSignerAddress

`getSignerAddress` is a read-only helper function used to recover a coordinator's address from it's signature.

```
/// @dev Recovers the address of a signer given a hash and signature.
/// @param hash Any 32 byte hash.
/// @param signature Proof that the hash has been signed by signer.
function getSignerAddress(bytes32 hash, bytes memory signature)
    public
    pure
    returns (address signerAddress);
```

### getTransactionHash

`getTransactionHash` is a read-only helper function that is used to calculate the hash of a 0x transaction.

```
    /// @dev Calculates the EIP712 hash of a 0x transaction using the domain separator of the `Exchange` contract.
    /// @param transaction 0x transaction containing salt, signerAddress, and data.
    /// @return EIP712 hash of the transaction with the domain separator of this contract.
    function getTransactionHash(ZeroExTransaction memory transaction)
        public
        view
        returns (bytes32 transactionHash)
```

### getCoordinatorApprovalHash

`getCoordinatorApprovalHash` is a read-only helper function that is used to calculate the hash of a coordinator's approval.

```
/// @dev Calculated the EIP712 hash of the Coordinator approval mesasage using the domain separator of this contract.
/// @param approval Coordinator approval message containing the transaction hash, transaction signature, and expiration of the approval.
/// @return EIP712 hash of the Coordinator approval message with the domain separator of this contract.
function getCoordinatorApprovalHash(CoordinatorApproval memory approval)
    public
    view
    returns (bytes32 approvalHash);
```

### decodeOrdersFromFillData

`decodeOrdersFromFillData` is a read-only helper function that is used to decode orders from any `Exchange` calldata that utilizes a fill function (such as a 0x transaction's data). Note this function will always return an empty array if invalid data or data from a non-fill function is passed in (such as a cancel function).

```
/// @dev Decodes the orders from Exchange calldata representing any fill method.
/// @param data Exchange calldata representing a fill method.
/// @return The orders from the Exchange calldata.
function decodeOrdersFromFillData(bytes memory data)
    public
    pure
    returns (LibOrder.Order[] memory orders);
```

## CoordinatorRegistry

The CoordinatorRegistry contract allows coordinators to register their [Standard Coordinator API](standard-coordinator-api) endpoints in a central contract for greater discoverability.

### setCoordinatorEndpoint

`setCoordinatorEndpoint` registers an endpoint string to the address of the sender.

```
/// @dev Called by a Coordinator operator to set the endpoint of their Coordinator.
/// @param coordinatorEndpoint endpoint of the Coordinator.
function setCoordinatorEndpoint(string calldata coordinatorEndpoint)
    external;
```

### getCoordinatorEndpoint

`getCoordinatorEndpoint` is a read-only function that retrieves the endpoint of a coordinator.

```
/// @dev Gets the endpoint for a Coordinator.
/// @param coordinatorOperator operator of the Coordinator endpoint.
function getCoordinatorEndpoint(address coordinatorOperator)
    external
    view
    returns (string memory coordinatorEndpoint);
```

# Signature Types

All signatures submitted to the `Coordinator` contract are represented as a byte array of arbitrary length, where the last byte (the "signature byte") specifies the signatures type. The signature type is popped from the signature byte array before validation. The allowed coordinator signature types are symmetric to the [Exchange signature types](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#signature-types), but do not include `Wallet`, `Validator`, and `PreSigned` types. The following signature types are supported:

| Signature byte | Signature type      |
| -------------- | ------------------- |
| 0x00           | [Illegal](#illegal) |
| 0x01           | [Invalid](#invalid) |
| 0x02           | [EIP712](#eip712)   |
| 0x03           | [EthSign](#ethsign) |

## Illegal

This is the default value of the signature byte. A transaction that includes an `Illegal` signature will be reverted. Therefore, users must explicitly specify a valid signature type.

## Invalid

An `Invalid` signature will always revert, much like the `Illegal` type. An invalid signature can always be recreated and is therefore offered explicitly. This signature type is largely used for testing purposes and to create symmetry with the `Exchange` signature types.

## EIP712

An `EIP712` signature is considered valid if the address recovered from calling [`ecrecover`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#ecrecover-usage) with the given hash and decoded `v`, `r`, `s` values is the same as the specified signer. In this case, the signature is encoded in the following way:

| Offset | Length | Contents            |
| ------ | ------ | ------------------- |
| 0x00   | 1      | v (always 27 or 28) |
| 0x01   | 32     | r                   |
| 0x21   | 32     | s                   |

## EthSign

An `EthSign` signature is considered valid if the address recovered from calling [`ecrecover`](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#ecrecover-usage) with the an EthSign-prefixed hash and decoded `v`, `r`, `s` values is the same as the specified signer.

The prefixed `msgHash` is calculated with:

```
string constant ETH_PERSONAL_MESSAGE = "\x19Ethereum Signed Message:\n32";
bytes32 msgHash = keccak256(abi.encodePacked(ETH_PERSONAL_MESSAGE, hash));
```

`v`, `r`, and `s` are encoded in the signature byte array using the same scheme as [EIP712 signatures](#eip712).

# Events

## Coordinator events

No events are emitted by the `Coordinator` contract. However, orders filled through the `Coordinator` contract can be detected by filtering the `Exchange` contracts events for a `senderAddress` that matches the `Coordinator` contract's address.

## CoordinatorRegistry events

### CoordinatorEndpointSet

A `CoordinatorEndpointSet` event is emitted when a coordinator endpoint is set.

```
/// @dev Emitted when a Coordinator endpoint is set.
event CoordinatorEndpointSet(
    address coordinatorOperator,
    string coordinatorEndpoint
);
```

# Reference Coordinator Server

## Design choices

The goals of this coordinator server implementation are to enable soft cancels and to enforce a selective delay between a fill request being received and approved. This implementation does _not_ explicitly mitigate trade collisions. However, this design has implicit benefits that may reduce the amount of collisions:

1. Fewer arbitrage opportunities should exist because of a maker's ability to quickly cancel orders. This reduces any opportunities to front-run trades.
1. All trade approvals are broadcast to all connected Websocket clients, which may be used as a signal to not attempt to fill already approved orders.

### Soft cancels

Typically, orders may only be cancelled using an on-chain method on the `Exchange` contract, such as `cancelOrder` or `cancelOrdersUpTo`. However, since coordinator orders may only be filled with an approval from their corresponding coordinator, they may be "soft cancelled" if the coordinator simply refuses to accept future fill requests for that order. This allows users to cancel outstanding orders quickly and freely without any on-chain transaction. Note that users must trust that the coordinator honor their cancel request until an order has expired or has been cancelled on-chain. However this process is made auditable with signed cancel receipts provided by the coordinator.

### Selective delay

The coordinator server adds a time delay between fill requests and approvals. This design decision has been made in order to give liquidity providers higher optionality and the ability to cancel stale orders more frequently (decreasing the chances of losing money on a trade). Simulations have shown this to decrease spreads, leading to a net benefit for both providers and consumers of liquidity. Note that this should also decrease profitability of coordinator orders for arbitraguers, which should indirectly decrease the number of trade collisions.

## Configuration

### SELECTIVE_DELAY_MS

The delay in milliseconds between the receipt of fill requests and fill approvals (default: 1000ms).

### EXPIRATION_DURATION_SECONDS

The amount of seconds an approval is valid for (default: 90 seconds). This parameter may be tweaked based off of the amount of congestion on the Ethereum network or the desired UX of your application. Users who are manually submitting transactions through a UI may need longer expiration times than a programmatic trader.

## State

The coordinator server must maintain state in order to determine the validity of transaction requests.

- The server must be able to identify orders for which it has issued a cancellation receipt (e.g by storing their hashes). This information can be discarded after the order has expired or been invalidated.
- For each order with at least one approved fill, the server must store the sum of approved `takerAssetFillAmount`s for each approved taker. This information can be discarded after the order has been filled or invalidated. Note that this information should persist even if outstanding approvals that involve the order have expired.
- For each order with at least one approved fill, the server must store any outstanding approval signatures for fill transactions that contain the order, as well as the corresponding expiration timestamp of each approval signature.
- The server must be able to identify transactions that it has already approved (e.g by storing its hash). This information can be discarded after the transaction has been executed or expired.

## Handling fills

_Note: `matchOrders` is not currently implemented in [0x-coordinator-server](https://github.com/0xProject/0x-coordinator-server). The server will not generate approval signatures for a `matchOrders` request. However, the Coordinator contract still requires approval signatures for any `matchOrders` transactions. As such, `matchOrders` transactions must be executed directly through the Exchange contract._

Fill transaction requests should be rejected under the following conditions:

- The transaction is in any way invalid (incorrect formatting, signature, unsupported function, etc).
- A transaction with the same hash has already been approved.
- The taker has requested fills for an included order with total `takerAssetFillAmount`s that exceed the `takerAssetAmount` of the order. This creates a cost for requesting fills and should prevent takers from frequently requesting fills that they do not intend on executing.
- An included order has been soft cancelled.

All other fill requests must be approved by the coordinator server exactly `SELECTIVE_DELAY_MS` after the request is received.

When a valid transaction request has been received, the coordinator server must broadcast a [`FILL_REQUEST_RECEIVED`](#fill_request_received) message to all connected Websocket clients. After a duration of `SELECTIVE_DELAY_MS`, the server should approve the fill request and simultaneously broadcast a [`FILL_REQUEST_APPROVED`](#fill_request_approved) message to all connected Websocket clients (if any orders contained in the transaction have not been soft cancelled in the mean time).

## Handling cancels

If a maker submits a valid signed 0x cancel transaction to the coordinator server (a transaction containing data for `cancelOrder` or `batchCancelOrders`), the server must no longer accept any future fill requests that contain the transaction orders. The server must respond with a signed cancel receipt and simultaneously broadcast a [`CANCEL_REQUEST_ACCEPTED`](#cancel_request_accepted) message to all connected Websocket clients.

# Standard Coordinator API

In order to ensure that trading clients know how to successfully request a signature from your coordinator server, it must strictly adhere to the following API specification. If it does not, you risk traders being unable to fill your orders.

## Errors

Unless the spec defines otherwise, errors to bad requests should respond with HTTP 4xx or status codes.

### Common error codes

| Code | Reason                               |
| ---- | ------------------------------------ |
| 400  | Bad Request – Invalid request format |
| 404  | Not found                            |
| 500  | Internal Server Error                |

### Error reporting format

```
{
    "code": 101,
    "reason": "Validation failed",
    "validationErrors": [
        {
            "field": "networkId",
            "code": 1003,
            "reason": "Requested networkId not supported by this coordinator"
        }
    ]
}
```

General error codes:

```
100 - Validation Failed
101 - Malformed JSON
102 - Request blocked
```

Validation error codes:

```
1000 - Required field
1001 - Incorrect format
1004 - Value out of range
1003 - Unsupported option
1004 - Included order already soft-cancelled
1005 - 0x transaction decoding failed
1006 - No coordinator orders included
1007 - Invalid 0x transaction signature
1008 - Only maker can cancel orders
1009 - Function call unsupported
1010 - Fill requests exceeded takerAssetAmount
1011 - 0x transaction already used
```

## Rest API

### Network Id

All requests should be able to specify a **?networkId** query param for all supported networks. For example:

```

curl https://api.coordinator.com/v1/request_transaction?networkId=1

```

If the query param is not provided, it should default to **1** (mainnet).

Some networks and their Ids:

| Network Id | Network Name        |
| ---------- | ------------------- |
| 1          | Mainnet             |
| 42         | Kovan               |
| 3          | Ropsten             |
| 4          | Rinkeby             |
| 50         | 0x Ganache snapshot |

If a certain network is not supported, the response should **400** as specified in the [error response](#error-response) section. For example:

```json
{
  "code": 100,
  "reason": "Validation failed",
  "validationErrors": [
    {
      "field": "networkId",
      "code": 1006,
      "reason": "Network id 42 is not supported"
    }
  ]
}
```

### GET /v1/configuration

This endpoint returns the specific configurations chosen by this Coordinator server.

#### Response

```json
{
  "expirationDurationSeconds": 90,
  "selectiveDelayMs": 1000,
  "supportedNetworkIds": [1, 42]
}
```

### POST /v1/request_transaction

Submit a signed 0x transaction encoding either a 0x fill or cancellation. If the 0x transaction encodes a fill, the sender is requesting a coordinator signature required to fill the order(s) on-chain. If the 0x transaction encodes an order(s) cancellation request, the sender is requesting the included order(s) to be soft-cancelled by the coordinator.

#### Payload

```json
{
  "signedTransaction": {
    "salt": "62799490869714002130337144204631467715674661702713158022807378407578488540214",
    "signerAddress": "0xe834ec434daba538cd1b9fe1582052b880bd7e63",
    "data": "0xb4be83d500000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000000000000000002a0000000000000000000000000e36ea790bc9d7ab70c55260c66d52b1eca985f84000000000000000000000000000000000000000000000000000000000000000000000000000000000000000078dc5d2d739606d31509c31d654056a45185ecb60000000000000000000000006ecbe1db9ef729cbe972c83fb886247691fb6beb0000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000ad78ebc5ac62000000000000000000000000000000000000000000000000000000de0b6b3a76400000000000000000000000000000000000000000000000000000de0b6b3a7640000000000000000000000000000000000000000000000000000000000005c879733d9cbe51a7e6b6c5e1f057d3d3ba3c1b57c8734fe0f523afa6250aff64aa9db37000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000001e00000000000000000000000000000000000000000000000000000000000000024f47261b000000000000000000000000034d402f14d58e001d8efbe6585051bf9706aa064000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000024f47261b000000000000000000000000025b8fe1de9daf8ba351890744ff28cf7dfa8f5e30000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000421c9e99c16367e608d75d943549b49fbeedcf58c84897b69daf0646e6184086a44107dda6e675826189c4bf509061880b5fe3e4707f449203249b98a39480bbee4803000000000000000000000000000000000000000000000000000000000000",
    "verifyingContractAddress": "0x48bacb9266a570d521063ef5dd96e61686dbe788",
    "signature": "0x1b36fca95c03d4e28b0ffb7796194841612005f0913e60f9da36f477a432e478240af81e8be55368b1a43f21266c6964e2b3d6c55b9d11dd5318ab76e63ff0c6f903"
  },
  "txOrigin": "0xe834ec434daba538cd1b9fe1582052b880bd7e63"
}
```

- `signedTransaction` - A signed [0x transaction](https://github.com/0xProject/0x-protocol-specification/blob/ad13141d9a2c6d93e06658d18c53e9f3d99442d4/v2/v2-specification.md#transactions) along with the `verifyingContractAddress` which is part of it's [EIP712 domain](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md#definition-of-domainseparator).
- `txOrigin` - The address that will eventually submit the Ethereum transaction executing this 0x transaction on-chain. This will be enforced by the `Coordinator` extension contract.

#### Response

**Fill request response:**

```json
{
  "signatures": [
    "0x1cc07d7ae39679690a91418d46491520f058e4fb14debdf2e98f2376b3970de8512ace44af0be6d1c65617f7aae8c2364ff63f241515ee1559c3eeecb0f671d9e903"
  ],
  "expirationTimeSeconds": "1552390014"
}
```

- `signatures` - the coordinator signatures required to submit the 0x transaction
- `expirationTimeSeconds` - when the signatures will expire and no longer be valid

Usually a single signature will be returned. Only when someone requests to batchFill multiple orders from the coordinator that were created with different supported feeRecipientAddresses, will this return a signature per feeRecipientAddress involved.

**Cancellation request response:**

```json
{
  "outstandingFillSignatures": [
    {
      "orderHash": "0xd1dc61f3e7e5f41d72beae7863487beea108971de678ca00d903756f842ef3ce",
      "approvalSignatures": [
        "0x1c7383ca8ebd6de8b5b20b1c2d49bea166df7dfe4af1932c9c52ec07334e859cf2176901da35f4480ceb3ab63d8d0339d851c31929c40d88752689b9a8a535671303"
      ],
      "expirationTimeSeconds": "1552390380",
      "takerAssetFillAmount": "100000000000000000000"
    }
  ],
  "cancellationSignatures": [
    "0x2ea3117a8ebd6de8b5b20b1c2d49bea166df7dfe4af1932c9c52ec07334e859cf2176901da35f4480ceb3ab63d8d0339d851c31929c40d88752689b9a855b5a7b401"
  ]
}
```

- `outstandingFillSignatures` - Information about the outstanding, unexpired signatures to fill the order(s) that have been soft-cancelled. Once they expire, the maker will have 100% certainty that their order can no longer be filled.
- `cancellationSignatures` - An approval signature of the cancellation 0x transaction submitted to the coordinator (with the expiration hard-coded to 0 -- although these never expire). These signatures can be used to prove that a soft-cancel was granted for these order(s).

### POST /v1/soft_cancels

Within the coordinator model, the Coordinator server is the source-of-truth when it comes to determining whether an order has been soft-cancelled. This endpoint can be used to query whether a set of orders have been soft-cancelled. The response returns the subset of orders that have been soft-cancelled.

#### Payload

```json
{
  "orderHashes": [
    "0xd1dc61f3e7e5f41d72beae7863487beea108971de678ca00d903756f842ef3ce", "0xabcc372e3812bba31a35958dc32d0beadf27595d51fb953709944559404e1b61"
  ]
}
```

#### Response

```json
{
  "orderHashes": [
    "0xabcc372e3812bba31a35958dc32d0beadf27595d51fb953709944559404e1b61"
  ]
}
```

## WebSocket API

### Transaction request notifications

#### Endpoint: `/v1/requests`

The WebSocket endpoint allows a client to subscribe to the following events:

#### FILL_REQUEST_RECEIVED

A fill request was received and is valid. The request has not yet been granted a coordinator signature. Depending on the server implementation, the request might need to wait for a selective delay before being issued a signature.

```json
{
  "type": "FILL_REQUEST_RECEIVED",
  "data": {
    "transactionHash": "0xef5c67ee7812fba2da35958dc32d0beadf27595d51fb953709944559404e1b6b"
  }
}
```

#### FILL_REQUEST_ACCEPTED

The fill request has been accepted and a signature issued. The corresponding 0x transaction can now be executed on-chain.

```json
{
  "type": "FILL_REQUEST_ACCEPTED",
  "data": {
    "functionName": "fillOrder",
    "orders": [
      {
        "makerAddress": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
        "takerAddress": "0x0000000000000000000000000000000000000000",
        "feeRecipientAddress": "0x78dc5d2d739606d31509c31d654056a45185ecb6",
        "senderAddress": "0x6ecbe1db9ef729cbe972c83fb886247691fb6beb",
        "makerAssetAmount": "100000000000000000000",
        "takerAssetAmount": "200000000000000000000",
        "makerFee": "1000000000000000000",
        "takerFee": "1000000000000000000",
        "expirationTimeSeconds": "1552396423",
        "salt": "66097384406870180066678463045003379626790660770396923976862707230261946348951",
        "makerAssetData": "0xf47261b000000000000000000000000034d402f14d58e001d8efbe6585051bf9706aa064",
        "takerAssetData": "0xf47261b000000000000000000000000025b8fe1de9daf8ba351890744ff28cf7dfa8f5e3",
        "exchangeAddress": "0x4f833a24e1f95d70f028921e27040ca56e09ab0b"
      }
    ],
    "txOrigin": "0x2ebac44a8b1218cabe972c83fb886247691fb6beb",
    "signedTransaction": {
      "salt": "1495510454898017617137906367124143312199249786920467590762356946976468764317",
      "signerAddress": "0xe834ec434daba538cd1b9fe1582052b880bd7e63",
      "data": "0xb4be83d500000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000000000000000002a0000000000000000000000000e36ea790bc9d7ab70c55260c66d52b1eca985f84000000000000000000000000000000000000000000000000000000000000000000000000000000000000000078dc5d2d739606d31509c31d654056a45185ecb60000000000000000000000006ecbe1db9ef729cbe972c83fb886247691fb6beb0000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000ad78ebc5ac62000000000000000000000000000000000000000000000000000000de0b6b3a76400000000000000000000000000000000000000000000000000000de0b6b3a7640000000000000000000000000000000000000000000000000000000000005c87b0879221cb37dcf690e02b0f9aecf44fcaa5ed9ce99697e86743795fa132596ff597000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000001e00000000000000000000000000000000000000000000000000000000000000024f47261b000000000000000000000000034d402f14d58e001d8efbe6585051bf9706aa064000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000024f47261b000000000000000000000000025b8fe1de9daf8ba351890744ff28cf7dfa8f5e30000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000421ce8e3c600d933423172b5021158a6be2e818613ff8e762d70ef490c752fd98a626a215f09f169668990414de75a53da221c294a3002f796d004827258b641876e03000000000000000000000000000000000000000000000000000000000000",
      "verifyingContractAddress": "0x48bacb9266a570d521063ef5dd96e61686dbe788",
      "signature": "0x1b36fca95c03d4e28b0ffb7796194841612005f0913e60f9da36f477a432e478240af81e8be55368b1a43f21266c6964e2b3d6c55b9d11dd5318ab76e63ff0c6f903"
    },
    "approvalSignatures": [
      "0x1b98e8b81249e49de6a76ae515108215621a2ad85909580de16a960280c97830eb2b417c7675775dc1ccaeca596deee5c8a0613058a48649e9b0ee2696b0cb975703"
    ],
    "approvalExpirationTimeSeconds": "1552395894"
  }
}
```

#### CANCEL_REQUEST_ACCEPTED

A cancellation request has been recevied and processed. The 0x orders included in the request have been soft-cancelled, and not additional fill signatures will be issued by the coordinator.

```json
{
  "type": "CANCEL_REQUEST_ACCEPTED",
  "data": {
    "orders": [
      {
        "makerAddress": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
        "takerAddress": "0x0000000000000000000000000000000000000000",
        "feeRecipientAddress": "0x78dc5d2d739606d31509c31d654056a45185ecb6",
        "senderAddress": "0x6ecbe1db9ef729cbe972c83fb886247691fb6beb",
        "makerAssetAmount": "100000000000000000000",
        "takerAssetAmount": "200000000000000000000",
        "makerFee": "1000000000000000000",
        "takerFee": "1000000000000000000",
        "expirationTimeSeconds": "1552396423",
        "salt": "77312126688742331532122416049113540452139616608135326468253415920865328290216",
        "makerAssetData": "0xf47261b000000000000000000000000034d402f14d58e001d8efbe6585051bf9706aa064",
        "takerAssetData": "0xf47261b000000000000000000000000025b8fe1de9daf8ba351890744ff28cf7dfa8f5e3",
        "exchangeAddress": "0x4f833a24e1f95d70f028921e27040ca56e09ab0b"
      }
    ],
    "transaction": {
      "salt": "30502121252152069340049118328369905364578815501441646224439039476012324011326",
      "signerAddress": "0xe36ea790bc9d7ab70c55260c66d52b1eca985f84",
      "data": "0xd46b02c30000000000000000000000000000000000000000000000000000000000000020000000000000000000000000e36ea790bc9d7ab70c55260c66d52b1eca985f84000000000000000000000000000000000000000000000000000000000000000000000000000000000000000078dc5d2d739606d31509c31d654056a45185ecb60000000000000000000000006ecbe1db9ef729cbe972c83fb886247691fb6beb0000000000000000000000000000000000000000000000056bc75e2d6310000000000000000000000000000000000000000000000000000ad78ebc5ac62000000000000000000000000000000000000000000000000000000de0b6b3a76400000000000000000000000000000000000000000000000000000de0b6b3a7640000000000000000000000000000000000000000000000000000000000005c87b087aaed1cee5db78cfae5b9be5a1a30c67fc3515aacfa536bea1aa33deb90e4d9a8000000000000000000000000000000000000000000000000000000000000018000000000000000000000000000000000000000000000000000000000000001e00000000000000000000000000000000000000000000000000000000000000024f47261b000000000000000000000000034d402f14d58e001d8efbe6585051bf9706aa064000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000024f47261b000000000000000000000000025b8fe1de9daf8ba351890744ff28cf7dfa8f5e300000000000000000000000000000000000000000000000000000000",
      "verifyingContractAddress": "0x48bacb9266a570d521063ef5dd96e61686dbe788"
    }
  }
}
```
