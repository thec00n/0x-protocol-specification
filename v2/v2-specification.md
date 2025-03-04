# 0x protocol 2.0.0 specification

## Table of contents

1.  [Architecture](#architecture)
1.  [Contracts](#contracts)
    1.  [Exchange](#exchange)
    1.  [AssetProxy](#assetproxy)
        1. [ERC20Proxy](#erc20proxy)
        1. [ERC721Proxy](#erc721proxy)
        1. [MultiAssetProxy](#multiassetproxy)
    1.  [AssetProxyOwner](#assetproxyowner)
1.  [Contract Interactions](#contract-interactions)
    1.  [Trade settlement](#trade-settlement)
    1.  [Upgrading the Exchange contract](#upgrading-the-exchange-contract)
    1.  [Upgrading the AssetProxyOwner contract](#upgrading-the-assetproxyowner-contract)
    1.  [Adding new AssetProxy contracts](#adding-new-assetproxy-contracts)
1.  [Orders](#orders)
    1.  [Message format](#order-message-format)
    1.  [Hashing an order](#hashing-an-order)
    1.  [Creating an order](#creating-an-order)
    1.  [Filling orders](#filling-orders)
    1.  [Cancelling orders](#cancelling-orders)
    1.  [Querying state of an order](#querying-state-of-an-order)
1.  [Transactions](#transactions)
    1.  [Message format](#transaction-message-format)
    1.  [Hash of a transaction](#hash-of-a-transaction)
    1.  [Creating a transaction](#creating-a-transaction)
    1.  [Executing a transaction](#executing-a-transaction)
    1.  [Filter contracts](#filter-contracts)
1.  [Signatures](#signatures)
    1.  [Validating signatures](#validating-signatures)
    1.  [Signature types](#signature-types)
1.  [Events](#events)
    1.  [Exchange events](#exchange-events)
    1.  [AssetProxy events](#assetproxy-events)
    1.  [AssetProxyOwner events](#assetproxyowner-events)
1.  [Types](#types)
1.  [Standard relayer API](#standard-relayer-api)
1.  [Miscellaneous](#miscellaneous)
    1.  [EIP712 usage](#eip712-usage)
    1.  [Optimizing calldata](#optimizing-calldata)
    1.  [ecrecover usage](#ecrecover-usage)
    1.  [Reentrancy protection](#reentrancy-protection)

# Architecture

0x protocol uses an approach we refer to as **off-chain order relay with on-chain settlement**. In this approach, cryptographically signed [orders](#orders) are broadcast off of the blockchain through any arbitrary communication channel; an interested counterparty may inject one or more of these [orders](#orders) into 0x protocol's [`Exchange`](#exchange) contract to execute and settle trades directly to the blockchain.

0x uses a modular system of Ethereum smart contracts which allows each component of the system to be upgraded via governance without affecting other components of the system and without causing active markets to be disrupted. Version 2 of 0x protocol further modularizes this contract pipeline through the introduction of [`AssetProxy`](#assetproxy) contracts, which allow new token standards, interfaces and payloads to be supported over time.

# Contracts

## Exchange

The Exchange contract contains the bulk of the business logic within 0x protocol. It is the entry point for:

1.  Filling [orders](#orders)
2.  Canceling [orders](#orders)
3.  Executing [transactions](#transactions)
4.  Validating [signatures](#signatures)
5.  Registering new [`AssetProxy`](#assetproxy) contracts into the system

## AssetProxy

The [`AssetProxy`](#assetproxy) contracts are responsible for:

1.  Decoding asset specific metadata contained within an order
2.  Performing the actual asset transfer
3.  Authorizing/unauthorizing Exchange contract addresses from calling the transfer methods on this [`AssetProxy`](#assetproxy)

In order to opt-in to using 0x protocol, users must approve an asset's associated [`AssetProxy`](#assetproxy) to transfer the asset on their behalf.

All [`AssetProxy`](#assetproxy) contracts have the following minimum interface:

```
contract IAuthorizable {

    /// @dev Gets all authorized addresses.
    /// @return Array of authorized addresses.
    function getAuthorizedAddresses()
        external
        view
        returns (address[]);

    /// @dev Authorizes an address.
    /// @param target Address to authorize.
    function addAuthorizedAddress(address target)
        external;

    /// @dev Removes authorizion of an address.
    /// @param target Address to remove authorization from.
    function removeAuthorizedAddress(address target)
        external;

    /// @dev Removes authorizion of an address.
    /// @param target Address to remove authorization from.
    /// @param index Index of target in authorities array.
    function removeAuthorizedAddressAtIndex(
        address target,
        uint256 index
    )
        external;
}

contract IAssetProxy is
    IAuthorizable
{

    /// @dev Transfers assets. Either succeeds or throws.
    /// @param assetData Byte array encoded for the respective asset proxy.
    /// @param from Address to transfer asset from.
    /// @param to Address to transfer asset to.
    /// @param amount Amount of asset to transfer.
    function transferFrom(
        bytes assetData,
        address from,
        address to,
        uint256 amount
    )
        external;

    /// @dev Gets the proxy id associated with the proxy address.
    /// @return Proxy id.
    function getProxyId()
        external
        view
        returns (uint8);
}
```

Currently, the protocol includes [`AssetProxy`](#assetproxy) contracts for ERC20 and ERC721 tokens.

### ERC20Proxy

The `ERC20Proxy` is responsible for transferring [ERC20 tokens](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md). Users must first approve this contract by calling the `approve` method on the token that will be exchanged. It is recommended that users approve a value of 2^256 -1. This minimizes the amount of times `approve` must be called, and also [increases efficiency](https://github.com/ethereum/EIPs/issues/717) for many ERC20 tokens.

This contract expects ERC20 [`assetData`](#assetdata) to be encoded using [ABIv2](http://solidity.readthedocs.io/en/latest/abi-spec.html) with the following 4 byte id:

```
// 0xf47261b0
bytes4 ERC20_SELECTOR = bytes4(keccak256("ERC20Token(address)"));
```

The data is then encoded as:

| Offset | Length | Contents                                        |
| ------ | ------ | ----------------------------------------------- |
| 0x00   | 4      | ERC20Proxy id (always 0xf47261b0)               |
| 0x04   | 32     | Address of ERC20 token, left padded with zeroes |

NOTE: The `ERC20Proxy` does not enforce strict length checks for [`assetData`](#assetdata), which means that extra data may be appended to this field with any arbitrary encoding. Any extra data will be ignored by the `ERC20Proxy` but may be used in external contracts interacting with the [`Exchange`](#exchange) contract. Relayers that do not desire this behavior should validate the length of all [`assetData`](#assetdata) fields contained in [orders](#orders) before acceptance.

The `ERC20Proxy` performs the transfer by calling the token's `transferFrom` method. The transaction will be reverted if the owner has insufficient balance or if the `ERC20Proxy` does not have sufficient allowance to perform the transfer.

### ERC721Proxy

The `ERC721Proxy` is responsible for transferring [ERC721 tokens](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md). Users must first approve this contract by calling the `approve` or `setApprovalForAll` methods on the token that will be exchanged. `setApprovalForAll` is highly recommended, because it allows the user to approve multiple `tokenIds` with a single transaction.

This contract expects ERC721 [`assetData`](#assetdata) to be encoded using [ABIv2](http://solidity.readthedocs.io/en/latest/abi-spec.html) with the following 4 byte id:

```
// 0x02571792
bytes4 ERC721_SELECTOR = bytes4(keccak256("ERC721Token(address,uint256)"));
```

The data is then encoded as:

| Offset | Length | Contents                                         |
| ------ | ------ | ------------------------------------------------ |
| 0x00   | 4      | ERC721 proxy id (always 0x02571792)              |
| 0x04   | 32     | Address of ERC721 token, left padded with zeroes |
| 0x24   | 32     | tokenId of ERC721 token                          |

NOTE: The `ERC721Proxy` does not enforce strict length checks for [`assetData`](#assetdata), which means that extra data may be appended to this field with any arbitrary encoding. Any extra data will be ignored by the `ERC721Proxy` but may be used in external contracts interacting with the [`Exchange`](#exchange) contract. Relayers that do not desire this behavior should validate the length of all [`assetData`](#assetdata) fields contained in [orders](#orders) before acceptance.

The `ERC721Proxy` performs the transfer by calling the token's `transferFrom` method. The transaction will be reverted if the owner has insufficient balance or if the `ERC721Proxy` is not approved to perform the transfer.

### MultiAssetProxy

The `MultiAssetProxy` expects an `amounts` (`uint256` array) and a `nestedAssetData` (array of [`asseData`](#assetdata) byte arrays) to be encoded within its own `assetData`. Each element of `amounts` corresponds to an element at the same index of `nestedAssetData`. The `MultiAssetProxy` will multiply each `amounts` element by the `amount` passed into `MultiAssetProxy.transferFrom` and then dispatch the corresponding element of `nestedAssetProxy` to the relevant [`AssetProxy`](#assetproxy) contract with the resulting `totalAmount`. This contract does not perform any `transferFrom` calls to assets directly and therefore does not require any additional user approvals.

This contract expects its `assetData` to be encoded using [ABIv2](http://solidity.readthedocs.io/en/latest/abi-spec.html) with the following 4 byte id:

```
// 0x94cfcdd7
MultiAsset(uint256[],bytes[])
```

The data is then encoded as:

| Offset   | Length | Contents                                |
| -------- | ------ | --------------------------------------- |
| 0x00     | 4      | MultiAsset proxy id (always 0x94cfcdd7) |
| 0x04     | 32     | Offset to `amounts`                     |
| 0x24     | 32     | Offset to `nestedAssetData`             |
| 0x44     | 32     | `amounts` length                        |
| 0x64     | a      | `amounts` contents                      |
| 0x84 + a | 32     | `nestedAssetData` length                |
| 0xA4 + a | b      | `nestedAssetData` contents              |

Each element of `nestedAssetData` must be encoded according to the specification of the corresponding `AssetProxy` contract. Note that initially, the `MultiAssetProxy` will not support dispatching a transfer to itself.

For more information on how the `MultiAssetProxy` works at a higher level, please refer to [ZEIP23](https://github.com/0xProject/ZEIPs/issues/23).

## AssetProxyOwner

The `AssetProxyOwner` contract is indirectly responsible for updating the [`Exchange`](#exchange) contracts that are allowed to call the transfer methods on each [`AssetProxy`](#assetproxy) contract. It is the only address that is allowed to call `addAuthorizedAddress` and `removeAuthorizedAddressAtIndex` on each [`AssetProxy`](#assetproxy). Any transaction created by the `AssetProxyOwner` must be proposed, confirmed, and then may be executed after a 2 week timelock. The only exception to this is that `removeAuthorizedAddressAtIndex` may be executed immediately, in case of security related bugs. The `AssetProxyOwner` may also call `transferOwnership`, allowing it to swap itself out with an upgraded contract.

# Contract Interactions

The diagrams provided below demonstrate interactions between various 0x smart contracts that make up the system. The arrow represents execution context within the EVM as a transaction is processed. Execution context is passed from the originating Ethereum account (circle) and between 0x's Ethereum smart contracts (rectangles) as they make external function calls into each other. Arrows are directed from the caller to the callee. Pseudocode is provided alongside each diagram to demonstrate what is happening at each step in the sequence of external function calls that occur during a given transaction.

## Trade settlement

A trade is initiated when an [order](#orders) is passed into the [`Exchange`](#exchange) contract. If the [order](#orders) is valid, the [`Exchange`](#exchange) contract will attempt to settle each leg of the trade by calling into the appropriate [`AssetProxy`](#assetproxy) contract for each asset being exchanged. Each [`AssetProxy`](#assetproxy) accepts and processes a payload of asset metadata and initiates a transfer. To simplify the trade settlement diagrams below, we assume that the orders being settled have zero fees.

### ERC20 <> ERC20

<div style="text-align: center;">
<img src="./img/0x_v2_trade_erc20_erc20.png" style="padding-bottom: 20px; padding-top: 20px;" width="80%" />
</div>

Transaction #1

1.  `Exchange.fillOrder(order, value)`
2.  `ERC20Proxy.transferFrom(assetData, from, to, value)`
3.  `ERC20Token(assetData.address).transferFrom(from, to, value)`
4.  ERC20Token: (revert on failure)
5.  ERC20Proxy: (revert on failure)
6.  `ERC20Proxy.transferFrom(assetData, from, to, value)`
7.  `ERC20Token(assetData.address).transferFrom(from, to, value)`
8.  ERC20Token: (revert on failure)
9.  ERC20Proxy: (revert on failure)
10. Exchange: (return [`FillResults`](#fillresults))

### ERC20 <> ERC721

<div style="text-align: center;">
<img src="./img/0x_v2_trade_erc20_erc721.png" style="padding-bottom: 20px; padding-top: 20px;" width="80%" />
</div>

Transaction #1

1.  `Exchange.fillOrder(order, value)`
2.  `ERC721Proxy.transferFrom(assetData, from, to, value)`
3.  `ERC721Token(assetData.address).transferFrom(from, to, assetData.tokenId)`
4.  ERC721Token: (revert on failure)
5.  ERC721Proxy: (revert on failure)
6.  `ERC20Proxy.transferFrom(assetData, from, to, value)`
7.  `ERC20Token(assetData.address).transferFrom(from, to, value)`
8.  ERC20Token: (revert on failure)
9.  ERC20Proxy: (revert on failure)
10. Exchange: (return [`FillResults`](#fillresults))

## Upgrading the Exchange contract

New [`Exchange`](#exchange) contracts can be added by calling `addAuthorizedAddress` on each [`AssetProxy`](#assetproxy) contract. Multiple `Exchange` contracts may exists at the same time, but typically older versions will be deprecated and removed by calling `removeAuthorizedAddressAtIndex` on each `AssetProxy` contract. Only the [`AssetProxyOwner`](#assetproxyowner) contract can call these methods.

## Upgrading the AssetProxyOwner contract

The [`AssetProxyOwner`](#assetproxyowner) contract can be upgraded by calling `transferOwnership` on each [`AssetProxy`](#assetproxy) contract, transferring ownership of the `AssetProxy` contracts to an upgraded contract. Future versions of the `AssetProxyOwner` will be able to execute transactions in batches, allowing upgrades to occur accross all `AssetProxy` contracts atomically.

## Adding new AssetProxy contracts

New [`AssetProxy`](#assetproxy) contracts may be added into the system by calling `registerAssetProxy` on the [`Exchange`](#exchange) contract. Only the [`AssetProxyOwner`](#assetproxyowner) contract may call this method.

# Orders

## Order message format

An order message consists of the following parameters:

| Parameter                       | Type    | Description                                                                                                                                     |
| ------------------------------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| makerAddress                    | address | Address that created the order.                                                                                                                 |
| takerAddress                    | address | Address that is allowed to fill the order. If set to 0, any address is allowed to fill the order.                                               |
| feeRecipientAddress             | address | Address that will receive fees when order is filled.                                                                                            |
| [senderAddress](#senderaddress) | address | Address that is allowed to call Exchange contract methods that affect this order. If set to 0, any address is allowed to call these methods.    |
| makerAssetAmount                | uint256 | Amount of makerAsset being offered by maker. Must be greater than 0.                                                                            |
| takerAssetAmount                | uint256 | Amount of takerAsset being bid on by maker. Must be greater than 0.                                                                             |
| makerFee                        | uint256 | Amount of ZRX paid to feeRecipient by maker when order is filled. If set to 0, no transfer of ZRX from maker to feeRecipient will be attempted. |
| takerFee                        | uint256 | Amount of ZRX paid to feeRecipient by taker when order is filled. If set to 0, no transfer of ZRX from taker to feeRecipient will be attempted. |
| expirationTimeSeconds           | uint256 | Timestamp in seconds at which order expires.                                                                                                    |
| [salt](#salt)                   | uint256 | Arbitrary number to facilitate uniqueness of the order's hash.                                                                                  |
| [makerAssetData](#assetdata)    | bytes   | ABIv2 encoded data that can be decoded by a specified proxy contract when transferring makerAsset.                                              |
| [takerAssetData](#assetdata)    | bytes   | ABIv2 encoded data that can be decoded by a specified proxy contract when transferring takerAsset.                                              |

### senderAddress

If the `senderAddress` of an order is not set to 0, only that address may call [`Exchange`](#exchange) contract methods that affect that order. See the [filter contracts examples](#filter-contracts) for more information.

### salt

An order's `salt` parameter has two main usecases:

- To ensure uniqueness within an order's hash.
- To be used in combination with [`cancelOrdersUpTo`](#cancelordersupto). When creating an order, the `salt` value _should_ be equal to the value of the current timestamp in milliseconds. This allows maker to create 1000 orders with the same parameters per second. Note that although this is part of the protocol specification, there is currently no way to enforce this usage and `salt` values should _not_ be relied upon as a source of truth.

### assetData

The `makerAssetData` and `takerAssetData` fields of an order contain information specific to that asset. These fields are encoded using [ABIv2](http://solidity.readthedocs.io/en/latest/abi-spec.html) with a 4 byte id that references the proxy that is intended to decode the data. See the [`ERC20Proxy`](#erc20proxy) and [`ERC721Proxy`](#erc721proxy) sections for the layouts of the `assetData` fields for each `AssetProxy` contract.

## Hashing an order

The hash of an order is used as a unique identifier of that order. An order is hashed according to the [EIP712 specification](#https://github.com/ethereum/EIPs/pull/712/files). See the [EIP712 Usage](#eip712-usage) section for information on how to calculate the required domain separator for hashing an order.

```
bytes32 constant EIP712_ORDER_SCHEMA_HASH = keccak256(abi.encodePacked(
    "Order(",
    "address makerAddress,",
    "address takerAddress,",
    "address feeRecipientAddress,",
    "address senderAddress,",
    "uint256 makerAssetAmount,",
    "uint256 takerAssetAmount,",
    "uint256 makerFee,",
    "uint256 takerFee,",
    "uint256 expirationTimeSeconds,",
    "uint256 salt,",
    "bytes makerAssetData,",
    "bytes takerAssetData",
    ")"
));

bytes32 orderHash = keccak256(abi.encodePacked(
    EIP191_HEADER,
    EIP712_DOMAIN_HASH,
    keccak256(abi.encodePacked(
        EIP712_ORDER_SCHEMA_HASH,
        bytes32(order.makerAddress),
        bytes32(order.takerAddress),
        bytes32(order.feeRecipientAddress),
        bytes32(order.senderAddress),
        order.makerAssetAmount,
        order.takerAssetAmount,
        order.makerFee,
        order.takerFee,
        order.expirationTimeSeconds,
        order.salt,
        keccak256(order.makerAssetData),
        keccak256(order.takerAssetData)
    ))
));
```

## Creating an order

An order may only be filled if it can be paired with an associated valid signature. Signatures are only validated the first time an order is filled. For later fills, no signature must be submitted. An order's hash must be signed with a [supported signature type](#signature-types).

## Filling orders

Orders can be filled by calling the following methods on the Exchange contract.

### fillOrder

This is the most basic way to fill an order. All of the other methods call `fillOrder` under the hood with additional logic. This function will attempt to fill the amount specified by the caller. However, if the remaining fillable amount is less than the amount specified, the remaining amount will be filled. Partial fills are allowed when filling orders.

`fillOrder` will revert under the following conditions:

- The caller of `fillOrder` is different from the `sender` specified in the order (unless `sender == address(0)`).
- The taker of `fillOrder` is different from the `taker` specified in the order (unless `taker == address(0)`).
- An invalid signature is submitted (this is only checked the first time an order is filled).
- The `makerAssetAmount` or `takerAssetAmount` specified in the order are equal to 0.
- The amount that the taker is attempting to fill is 0.
- The order has expired.
- The order has been cancelled.
- The order has already been fully filled.
- Filling the order results in a rounding error > 0.1% of the `takerAssetAmount` that would otherwise be filled.
- Any transfers associated with the fill fails.
- The amount the taker is attempting to fill multiplied by the `makerAssetAmount` is greater than 256 bits.
- The amount the taker is attempting to fill multiplied by the `makerFee` is greater than 256 bits.
- The amount the taker is attempting to fill multiplied by the `takerFee` is greater than 256 bits.
- [Reentrancy](#reentrancy-protection) is attempted to any function within the `Exchange` contract that contains a mutex.

If successful, `fillOrder` will emit a [`Fill`](#fill) event. If the transaction does not revert, a [`FillResults`](#fillresults) instance will be returned.

```
/// @dev Fills the input order.
/// @param order Order struct containing order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signature Proof that order has been created by maker.
/// @return Amounts filled and fees paid by maker and taker.
function fillOrder(
    Order memory order,
    uint256 takerAssetFillAmount,
    bytes memory signature
)
    public
    returns (FillResults memory fillResults);
```

### fillOrKillOrder

`fillOrKillOrder` behaves almost exactly the same as [`fillOrder`](#fillorder). However, the transaction will revert if the amount specified is not filled exactly.

```
/// @dev Fills the input order. Reverts if exact takerAssetFillAmount not filled.
/// @param order Order struct containing order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signature Proof that order has been created by maker.
function fillOrKillOrder(
    Order memory order,
    uint256 takerAssetFillAmount,
    bytes memory signature
)
    public
    returns (FillResults memory fillResults);
```

### fillOrderNoThrow

`fillOrderNoThrow` also behaves very similary to [`fillOrder`](#fillorder). However, the transaction will never revert and will instead return a [`FillResults`](#fillresults) instance that contains all 0 values. This is useful when calling the batch methods listed below, where a user may not want an entire transaction to fail when a single fill is reverted.

```
/// @dev Fills an order with specified parameters and ECDSA signature.
///      Returns false if the transaction would otherwise revert.
/// @param order Order struct containing order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signature Proof that order has been created by maker.
/// @return Amounts filled and fees paid by maker and taker.
function fillOrderNoThrow(
    Order memory order,
    uint256 takerAssetFillAmount,
    bytes memory signature
)
    public
    returns (FillResults memory fillResults);
```

### batchFillOrders

`batchFillOrders` calls [`fillOrder`](#fillorder) sequentially for each provided order, amount, and signature.

```
/// @dev Synchronously executes multiple calls of fillOrder.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmounts Array of desired amounts of takerAsset to sell in orders.
/// @param signatures Proofs that orders have been created by makers.
/// @return Amounts filled and fees paid by makers and taker.
///         NOTE: makerAssetFilledAmount and takerAssetFilledAmount may include amounts filled of different assets.
function batchFillOrders(
    Order[] memory orders,
    uint256[] memory takerAssetFillAmounts,
    bytes[] memory signatures
)
    public;
    returns (FillResults memory totalFillResults)
```

### batchFillOrKillOrders

`batchFillOrKillOrders` calls [`fillOrKillOrder`](#fillorkillorder) sequentially for each provided order, amount, and signature.

```
/// @dev Synchronously executes multiple calls of fillOrKill.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmounts Array of desired amounts of takerAsset to sell in orders.
/// @param signatures Proofs that orders have been created by makers.
/// @return Amounts filled and fees paid by makers and taker.
///         NOTE: makerAssetFilledAmount and takerAssetFilledAmount may include amounts filled of different assets.
function batchFillOrKillOrders(
    Order[] memory orders,
    uint256[] memory takerAssetFillAmounts,
    bytes[] memory signatures
)
    public;
    returns (FillResults memory totalFillResults)
```

### batchFillOrdersNoThrow

`batchFillOrdersNoThrow` calls [`fillOrderNoThrow`](#fillordernothrow) sequentially for each provided order, amount, and signature.

```
/// @dev Synchronously executes multiple calles of fillOrderNoThrow.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmounts Array of desired amounts of takerAsset to sell in orders.
/// @param signatures Proofs that orders have been created by makers.
/// @return Amounts filled and fees paid by makers and taker.
///         NOTE: makerAssetFilledAmount and takerAssetFilledAmount may include amounts filled of different assets.
function batchFillOrdersNoThrow(
    Order[] memory orders,
    uint256[] memory takerAssetFillAmounts,
    bytes[] memory signatures
)
    public;
    returns (FillResults memory totalFillResults)
```

### marketSellOrders

`marketSellOrders` calls [`fillOrder`](#fillorder) sequentially for each provided order and signature until the total `takerAssetAmount` has been sold by the taker. If successful, `marketSellOrders` returns a [`FillResults`](#fillresults) instance containing the cumulative amounts filled and fees paid.

Note that `marketSellOrders` assumes that the `takerAssetData` is equal for each order. For any order passed in after the first, the `takerAssetData` byte array will be ignored (allowing null byte arrays to be passed in). If an order was intended to use a different `takerAssetData` field, the fill will fail at signature validation.

```
/// @dev Synchronously executes multiple calls of fillOrder until total amount of takerAsset is sold by taker.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signatures Proofs that orders have been created by makers.
/// @return Amounts filled and fees paid by makers and taker.
function marketSellOrders(
    Order[] memory orders,
    uint256 takerAssetFillAmount,
    bytes[] memory signatures
)
    public
    returns (FillResults memory totalFillResults);
```

### marketSellOrdersNoThrow

`marketSellOrdersNoThrow` calls [`fillOrderNoThrow`](#fillordernothrow) sequentially for each provided order and signature until the total `takerAssetAmount` has been sold by the taker. If successful, `marketSellOrdersNoThrow` returns a [`FillResults`](#fillresults) instance containing the cumulative amounts filled and fees paid.

Note that `marketSellOrdersNoThrow` assumes that the `takerAssetData` is equal for each order. For any order passed in after the first, the `takerAssetData` byte array will be ignored (allowing null byte arrays to be passed in). If an order was intended to use a different `takerAssetData` field, the fill will fail at signature validation.

```
/// @dev Synchronously executes multiple calls of fillOrder until total amount of takerAsset is sold by taker.
///      Returns false if the transaction would otherwise revert.
/// @param orders Array of order specifications.
/// @param takerAssetFillAmount Desired amount of takerAsset to sell.
/// @param signatures Proofs that orders have been signed by makers.
/// @return Amounts filled and fees paid by makers and taker.
function marketSellOrdersNoThrow(
    Order[] memory orders,
    uint256 takerAssetFillAmount,
    bytes[] memory signatures
)
    public
    returns (FillResults memory totalFillResults);
```

### marketBuyOrders

`marketBuyOrders` calls [`fillOrder`](#fillorder) sequentially for each provided order and signature until the total `makerAssetAmount` has been bought by the taker. If successful, `marketBuyOrders` returns a [`FillResults`](#fillresults) instance containing the cumulative amounts filled and fees paid.

Note that `marketBuyOrders` assumes that the `makerAssetData` is equal for each order. For any order passed in after the first, the `makerAssetData` byte array will be ignored (allowing null byte arrays to be passed in). If an order was intended to use a different `makerAssetData` field, the fill will fail at signature validation.

```
/// @dev Synchronously executes multiple calls of fillOrder until total amount of makerAsset is bought by taker.
/// @param orders Array of order specifications.
/// @param makerAssetFillAmount Desired amount of makerAsset to buy.
/// @param signatures Proofs that orders have been signed by makers.
/// @return Amounts filled and fees paid by makers and taker.
function marketBuyOrders(
    Order[] memory orders,
    uint256 makerAssetFillAmount,
    bytes[] memory signatures
)
    public
    returns (FillResults memory totalFillResults);
```

### marketBuyOrdersNoThrow

`marketBuyOrdersNoThrow` calls [`fillOrderNoThrow`](#fillordernothrow) sequentially for each provided order and signature until the total `makerAssetAmount` has been bought by `taker`. If successful, `marketBuyOrdersNoThrow` returns a [`FillResults`](#fillresults) instance containing the cumulative amounts filled and fees paid.

```
/// @dev Synchronously executes multiple fill orders in a single transaction until total amount is bought by taker.
///      Returns false if the transaction would otherwise revert.
/// @param orders Array of order specifications.
/// @param makerAssetFillAmount Desired amount of makerAsset to buy.
/// @param signatures Proofs that orders have been signed by makers.
/// @return Amounts filled and fees paid by makers and taker.
function marketBuyOrdersNoThrow(
    Order[] memory orders,
    uint256 makerAssetFillAmount,
    bytes[] memory signatures
)
    public
    returns (FillResults memory totalFillResults);
```

## matchOrders

Two orders that represent a bid and an ask for the same token pair may be matched together if the spread between their respective prices is negative. This is satified by the following equation:

```
(leftOrder.makerAssetAmount * rightOrder.makerAssetAmount) >= (leftOrder.takerAssetAmount * rightOrder.takerAssetAmount)
```

The caller of `matchOrders` is considered the taker for each order. The taker will pay the `takerFee` for each order, but will also receive the spread between both orders. The spread is always denominated in terms of the left order's makerAsset. No balance is required in order to call `matchOrders`, and the taker never holds intermediate balances of either asset.

`matchOrders` will revert if either order fails the validation checks for [fillOrder](#fillOrder). Note that `matchOrders` assumes that `rightOrder.makerAssetData == leftOrder.takerAssetData` and `rightOrder.takerAssetData == leftOrder.makerAssetData`, allowing null byte arrays to be passed in for both assetData fields of `rightOrder`. If other assetData fields were part of the original `rightOrder`, this function will fail when validating the signature of the `rightOrder`.

If successful, `matchOrders` will emit a [Fill](#fill) event for each matched order.

```
/// @dev Match two complementary orders that have a profitable spread.
///      Each order is filled at their respective price point. However, the calculations are
///      carried out as though the orders are both being filled at the right order's price point.
///      The profit made by the left order goes to the taker (who matched the two orders).
/// @param leftOrder First order to match.
/// @param rightOrder Second order to match.
/// @param leftSignature Proof that order was created by the left maker.
/// @param rightSignature Proof that order was created by the right maker.
/// @return matchedFillResults Amounts filled and fees paid by maker and taker of matched orders.
function matchOrders(
    Order memory leftOrder,
    Order memory rightOrder,
    bytes memory leftSignature,
    bytes memory rightSignature
)
    public
    returns (MatchedFillResults memory matchedFillResults);
```

## Cancelling orders

### cancelOrder

`cancelOrder` cancels the specified order. Partial cancels are not allowed.

`cancelOrder` will revert under the following conditions:

- The `makerAssetAmount` or `takerAssetAmount` specified in the order are equal to 0.
- The caller of `cancelOrder` is different from the `senderAddress` specified in the order (unless `senderAddress == address(0)`).
- The maker of the order has not authorized the cancel, either by calling `cancelOrder` through an Ethereum transaction or a [0x transaction](#transactions).
- The order has expired.
- The order has already been cancelled.

If successful, `cancelOrder` will emit a [`Cancel`](#cancel) event.

```
/// @dev After calling, the order can not be filled anymore.
/// @param order Order struct containing order specifications.
/// @return True if the order state changed to cancelled.
///         False if the transaction was already cancelled or expired.
function cancelOrder(Order memory order)
    public;
```

### cancelOrdersUpTo

`cancelOrdersUpTo` invalidates all orders created by the maker that have 1) a `salt` value that is less than or equal to the specified `targetOrderEpoch` and 2) that have a [`senderAddress`](#senderaddress) value equal to the caller (or null address if the caller is the maker). `cancelOrdersUpTo` also updates the current `orderEpoch`. This function will revert if `targetOrderEpoch` is less than or equal to the current `orderEpoch`. If successful, `cancelOrdersUpTo` will emit a [`CancelUpTo`](#cancelupto) event.

```
/// @dev Cancels all orders created by makerAddress with a salt less than or equal to the targetOrderEpoch
///      and senderAddress equal to msg.sender (or null address if msg.sender == makerAddress).
/// @param targetOrderEpoch Orders created with a salt less or equal to this value will be cancelled.
function cancelOrdersUpTo(uint256 targetOrderEpoch)
    external;
```

### batchCancelOrders

`batchCancelOrders` calls `cancelOrder` sequentially for each provided order.

```
/// @dev Synchronously executed musltiple calls of cancelOrder.
/// @param orders Array of order specifications.
function batchCancelOrders(Order[] memory orders)
    public;
```

## Querying state of an order

### filled

The Exchange contract contains a mapping that records the nominal amount of an order's `takerAssetAmount` that has already been filled. This mapping is updated each time an order is successfully filled, allowing for partial fills.

```
// Mapping of orderHash => amount of takerAsset already bought by maker
mapping (bytes32 => uint256) public filled;
```

### cancelled

The Exchange contract contains a mapping that records if an order has been cancelled.

```
// Mapping of orderHash => cancelled
mapping (bytes32 => bool) public cancelled;
```

### orderEpoch

The Exchange contract contains a mapping that specifies the `orderEpoch` for a given `makerAddress`/[`senderAddress`](#senderaddress) pair, which invalidates all orders containing that pair that contain a salt value less than or equal to the current `orderEpoch`.

```
// Mapping of makerAddress => senderAddress => lowest salt an order can have in order to be fillable
// Orders with specified senderAddress and with a salt less than their epoch to are considered cancelled
mapping (address => mapping (address => uint256)) public orderEpoch;
```

### getOrderInfo

`getOrderInfo` is a public method that returns the state, hash, and amount of an order that has already been filled as an [OrderInfo](#orderinfo) instance:

```
/// @dev Gets information about an order: status, hash, and amount filled.
/// @param order Order to gather information on.
/// @return OrderInfo Information about the order and its state.
///         See LibOrder.OrderInfo for a complete description.
function getOrderInfo(Order memory order)
    public
    view
    returns (OrderInfo memory orderInfo);
```

### getOrdersInfo

`getOrdersInfo` calls [`getOrderInfo`](#getorderinfo) sequentially for each provided order.

```
/// @dev Fetches information for all passed in orders.
/// @param orders Array of order specifications.
/// @return Array of OrderInfo instances that correspond to each order.
function getOrdersInfo(LibOrder.Order[] memory orders)
    public
    view
    returns (LibOrder.OrderInfo[] memory);
```

# Transactions

Transaction messages exist for the purpose of calling methods on the [`Exchange`](#exchange) contract in the context of another address (see [ZEIP18](https://github.com/0xProject/ZEIPs/issues/18)). This is especially useful for implementing [filter contracts](#filter-contracts).

## Transaction message format

| Parameter     | Type    | Description                                                                      |
| ------------- | ------- | -------------------------------------------------------------------------------- |
| signerAddress | address | Address of transaction signer                                                    |
| salt          | uint256 | Arbitrary number to facilitate uniqueness of the transactions's hash.            |
| data          | bytes   | The calldata that is to be executed. This must call an Exchange contract method. |

## Hash of a transaction

The hash of a transaction is used as a unique identifier for that transaction. A transaction is hashed according to the [EIP712 specification](#https://github.com/ethereum/EIPs/pull/712/files). See the [EIP712 Usage](#eip712-usage) section for information on how to calculate the required domain separator for hashing an order.

```
// Hash for the EIP712 ZeroEx Transaction Schema
bytes32 constant internal EIP712_ZEROEX_TRANSACTION_SCHEMA_HASH = keccak256(abi.encodePacked(
    "ZeroExTransaction(",
    "uint256 salt,",
    "address signerAddress,",
    "bytes data",
    ")"
));

bytes32 transactionHash = keccak256(abi.encodePacked(
    EIP191_HEADER,
    EIP712_DOMAIN_HASH,
    keccak256(abi.encodePacked(
        EIP712_ZEROEX_TRANSACTION_SCHEMA_HASH,
        salt,
        bytes32(signerAddress),
        keccak256(data)
    ))
));
```

## Creating a transaction

A transaction may only be executed if it can be paired with an associated valid signature. A transaction's hash must be signed with a [supported signature type](#signature-types).

## Executing a transaction

A transaction may only be executed by calling the `executeTransaction` method of the Exchange contract. `executeTransaction` attempts to execute any function on the Exchange contract in the context of the transaction signer (rather than `msg.sender`).

`executeTransaction` will revert under the following conditions:

- Reentrancy is attempted (e.g `executeTransaction` calls `executeTransaction` again).
- A transaction with an equivalent hash has already been executed.
- An invalid signature is submitted.
- The execution of the provided data reverts.

```
/// @dev Executes an exchange method call in the context of signer.
/// @param salt Arbitrary number to ensure uniqueness of transaction hash.
/// @param signerAddress Address of transaction signer.
/// @param data AbiV2 encoded calldata.
/// @param signature Proof that transaction has been signed by signer.
function executeTransaction(
    uint256 salt,
    address signerAddress,
    bytes data,
    bytes signature
)
    external;
```

## Filter contracts

A filter contract is intended to add or remove logic to how orders are executed. An order may be tied to a specific filter contract by setting its [`senderAddress`](#senderaddress) to the address of the desired filter contract.

Here are some simple examples that demonstrate how filter contracts may be used:

### ExchangeWrapper

This contract does not add any additional logic to how orders are filled or cancelled. It is primarily intended to show the flow of data from a filter contract to the [`Exchange`](#exchange) contract. It is important to note that orders that specify this contract as the [`senderAddress`](#senderaddress) would _only_ be able to use the methods defined in this filter contract. Those orders could not be filled or cancelled with any other [`Exchange`](#exchange) methods.

```
contract ExchangeWrapper {

    // Exchange contract.
    // solhint-disable-next-line var-name-mixedcase
    IExchange internal EXCHANGE;

    constructor (address _exchange)
        public
    {
        EXCHANGE = IExchange(_exchange);
    }

    /// @dev Cancels all orders created by sender with a salt less than or equal to the targetOrderEpoch
    ///      and senderAddress equal to this contract.
    /// @param targetOrderEpoch Orders created with a salt less or equal to this value will be cancelled.
    /// @param salt Arbitrary value to gaurantee uniqueness of 0x transaction hash.
    /// @param makerSignature Proof that maker wishes to call this function with given params.
    function cancelOrdersUpTo(
        uint256 targetOrderEpoch,
        uint256 salt,
        bytes makerSignature
    )
        external
    {
        address makerAddress = msg.sender;

        // Encode arguments into byte array.
        bytes memory data = abi.encodeWithSelector(
            EXCHANGE.cancelOrdersUpTo.selector,
            targetOrderEpoch
        );

        // Call `cancelOrdersUpTo` via `executeTransaction`.
        EXCHANGE.executeTransaction(
            salt,
            makerAddress,
            data,
            makerSignature
        );
    }

    /// @dev Fills an order using `msg.sender` as the taker.
    /// @param order Order struct containing order specifications.
    /// @param takerAssetFillAmount Desired amount of takerAsset to sell.
    /// @param salt Arbitrary value to gaurantee uniqueness of 0x transaction hash.
    /// @param orderSignature Proof that order has been created by maker.
    /// @param takerSignature Proof that taker wishes to call this function with given params.
    function fillOrder(
        LibOrder.Order memory order,
        uint256 takerAssetFillAmount,
        uint256 salt,
        bytes memory orderSignature,
        bytes memory takerSignature
    )
        public
    {
        address takerAddress = msg.sender;

        // Encode arguments into byte array.
        bytes memory data = abi.encodeWithSelector(
            EXCHANGE.fillOrder.selector,
            order,
            takerAssetFillAmount,
            orderSignature
        );

        // Call `fillOrder` via `executeTransaction`.
        EXCHANGE.executeTransaction(
            salt,
            takerAddress,
            data,
            takerSignature
        );
    }
}
```

### Whitelist

This contract is a bit more complex than the last. Orders that specify this contract as the [`senderAddress`](#senderaddress) could only be filled if both the maker and taker of the order are on a whitelist created by the filter contract owner.

This contract also makes use of the [`Validator`](#validator) signature type. Rather than requiring the taker to sign a 0x transaction _and_ an Ethereum transaction to call this contract, this contract makes use of `tx.origin` to only require a signed Ethereum transaction (this may have dangerous consequences without extra measures and is only intended to be an example).

```
contract Whitelist is
    Ownable
{

    // Mapping of address => whitelist status.
    mapping (address => bool) public isWhitelisted;

    // Exchange contract.
    // solhint-disable var-name-mixedcase
    IExchange internal EXCHANGE;
    bytes internal TX_ORIGIN_SIGNATURE;
    // solhint-enable var-name-mixedcase

    byte constant internal VALIDATOR_SIGNATURE_BYTE = "\x06";

    constructor (address _exchange)
        public
    {
        EXCHANGE = IExchange(_exchange);
        TX_ORIGIN_SIGNATURE = abi.encodePacked(address(this), VALIDATOR_SIGNATURE_BYTE);
    }

    /// @dev Adds or removes an address from the whitelist.
    /// @param target Address to add or remove from whitelist.
    /// @param isApproved Whitelist status to assign to address.
    function updateWhitelistStatus(
        address target,
        bool isApproved
    )
        external
        onlyOwner
    {
        isWhitelisted[target] = isApproved;
    }

    /// @dev Verifies signer is same as signer of current Ethereum transaction.
    ///      NOTE: This function can currently be used to validate signatures coming from outside of this contract.
    ///      Extra safety checks can be added for a production contract.
    /// @param signerAddress Address that should have signed the given hash.
    /// @param signature Proof of signing.
    /// @return Validity of order signature.
    // solhint-disable no-unused-vars
    function isValidSignature(
        bytes32 hash,
        address signerAddress,
        bytes signature
    )
        external
        view
        returns (bool isValid)
    {
        // solhint-disable-next-line avoid-tx-origin
        return signerAddress == tx.origin;
    }
    // solhint-enable no-unused-vars

    /// @dev Fills an order using `msg.sender` as the taker.
    ///      The transaction will revert if both the maker and taker are not whitelisted.
    ///      Orders should specify this contract as the `senderAddress` in order to gaurantee
    ///      that both maker and taker have been whitelisted.
    /// @param order Order struct containing order specifications.
    /// @param takerAssetFillAmount Desired amount of takerAsset to sell.
    /// @param salt Arbitrary value to gaurantee uniqueness of 0x transaction hash.
    /// @param orderSignature Proof that order has been created by maker.
    function fillOrderIfWhitelisted(
        LibOrder.Order memory order,
        uint256 takerAssetFillAmount,
        uint256 salt,
        bytes memory orderSignature
    )
        public
    {
        address takerAddress = msg.sender;

        // This contract must be the entry point for the transaction.
        require(
            // solhint-disable-next-line avoid-tx-origin
            takerAddress == tx.origin,
            "INVALID_SENDER"
        );

        // Check if maker is on the whitelist.
        require(
            isWhitelisted[order.makerAddress],
            "MAKER_NOT_WHITELISTED"
        );

        // Check if taker is on the whitelist.
        require(
            isWhitelisted[takerAddress],
            "TAKER_NOT_WHITELISTED"
        );

        // Encode arguments into byte array.
        bytes memory data = abi.encodeWithSelector(
            EXCHANGE.fillOrder.selector,
            order,
            takerAssetFillAmount,
            orderSignature
        );

        // Call `fillOrder` via `executeTransaction`.
        EXCHANGE.executeTransaction(
            salt,
            takerAddress,
            data,
            TX_ORIGIN_SIGNATURE
        );
    }
}
```

# Signatures

## Validating signatures

The `Exchange` contract includes a public method `isValidSignature` for validating signatures. This method has the following interface:

```
/// @dev Verifies that a signature is valid.
/// @param hash Message hash that is signed.
/// @param signerAddress Address of signer.
/// @param signature Proof of signing.
/// @return Validity of order signature.
function isValidSignature(
    bytes32 hash,
    address signerAddress,
    bytes memory signature
)
    public
    view
    returns (bool isValid);
```

## Signature Types

All signatures submitted to the Exchange contract are represented as a byte array of arbitrary length, where the last byte (the "signature byte") specifies the signatures type. The signature type is popped from the signature byte array before validation. The following signature types are supported within the protocol:

| Signature byte | Signature type          |
| -------------- | ----------------------- |
| 0x00           | [Illegal](#illegal)     |
| 0x01           | [Invalid](#invalid)     |
| 0x02           | [EIP712](#eip712)       |
| 0x03           | [EthSign](#ethsign)     |
| 0x04           | [Wallet](#wallet)       |
| 0x05           | [Validator](#validator) |
| 0x06           | [PreSigned](#presigned) |

### Illegal

This is the default value of the signature byte. A transaction that includes an Illegal signature will be reverted. Therefore, users must explicitly specify a valid signature type.

### Invalid

An `Invalid` signature always returns false. An invalid signature can always be recreated and is therefore offered explicitly. This signature type is largely used for testing purposes.

### EIP712

An `EIP712` signature is considered valid if the address recovered from calling [`ecrecover`](#ecrecover-usage) with the given hash and decoded `v`, `r`, `s` values is the same as the specified signer. In this case, the signature is encoded in the following way:

| Offset | Length | Contents            |
| ------ | ------ | ------------------- |
| 0x00   | 1      | v (always 27 or 28) |
| 0x01   | 32     | r                   |
| 0x21   | 32     | s                   |

### EthSign

An `EthSign` signature is considered valid if the address recovered from calling [`ecrecover`](#ecrecover-usage) with the an EthSign-prefixed hash and decoded `v`, `r`, `s` values is the same as the specified signer.

The prefixed `msgHash` is calculated with:

```
string constant ETH_PERSONAL_MESSAGE = "\x19Ethereum Signed Message:\n32";
bytes32 msgHash = keccak256(abi.encodePacked(ETH_PERSONAL_MESSAGE, hash));
```

`v`, `r`, and `s` are encoded in the signature byte array using the same scheme as [EIP712 signatures](#EIP712).

### Wallet

The `Wallet` signature type allows a contract to trade on behalf of any other address(es) by defining its own signature validation function. When used with order signing, the `Wallet` contract _is_ the `maker` of the order and should hold any assets that will be traded. When using this signature type, the [`Exchange`](#exchange) contract makes a `STATICCALL` to the `Wallet` contract's `isValidSignature` method, which means that signature verifcation will fail and revert if the `Wallet` attempts to update state. This contract should have the following interface:

```
contract IWallet {

    /// @dev Verifies that a signature is valid.
    /// @param hash Message hash that is signed.
    /// @param signature Proof of signing.
    /// @return Validity of order signature.
    function isValidSignature(
        bytes32 hash,
        bytes signature
    )
        external
        view
        returns (bytes4 magicValue);
}
```

A `Wallet` contract's `isValidSignature` method must return the following magic value if successful:

```solidity
// 0xb0671381
bytes4 WALLET_MAGIC_VALUE = bytes4(keccak256("isValidWalletSignature(bytes32,address,bytes)"));
```

Note when using this method to sign orders: although it can be useful to allow the validity of signatures to be determined by some state stored on the blockchain, it should be noted that the signature will only be checked the first time an order is filled. Therefore, the signature cannot be later invalidated by updating the associates state.

### Validator

The `Validator` signature type allows an address to delegate signature verification to any other address. The `Validator` contract must first be approved by calling the `setSignatureValidatorApproval` method:

```
// Mapping of signer => validator => approved
mapping (address => mapping (address => bool)) public allowedValidators;

/// @dev Approves/unnapproves a Validator contract to verify signatures on signer's behalf.
/// @param validatorAddress Address of Validator contract.
/// @param approval Approval or disapproval of  Validator contract.
function setSignatureValidatorApproval(
    address validatorAddress,
    bool approval
)
    external;
```

The `setSignatureValidatorApproval` method emits a [`SignatureValidatorApproval`](#signaturevalidatorapprovalset) event when executed.

A Validator signature is then encoded as:

| Offset   | Length | Contents                   |
| -------- | ------ | -------------------------- |
| 0x00     | x      | signature                  |
| 0x00 + x | 20     | Validator contract address |

A Validator contract must have the following interface:

```
contract IValidator {

    /// @dev Verifies that a signature is valid.
    /// @param hash Message hash that is signed.
    /// @param signerAddress Address that should have signed the given hash.
    /// @param signature Proof of signing.
    /// @return Validity of order signature.
    function isValidSignature(
        bytes32 hash,
        address signerAddress,
        bytes signature
    )
        external
        view
        returns (bytes4 magicValue);
}
```

A `Validator` contract's `isValidSignature` method must return the following magic value if successful:

```solidity
// 0x42b38674
bytes4 VALIDATOR_MAGIC_VALUE = bytes4(keccak256("isValidValidatorSignature(address,bytes32,address,bytes)"));
```

The signature is validated by calling the `Validator` contract's `isValidSignature` method. When using this signature type, the [`Exchange`](#exchange) contract makes a `STATICCALL` to the `Validator` contract's `isValidSignature` method, which means that signature verifcation will fail and revert if the `Validator` attempts to update state.

```
// Pop last 20 bytes off of signature byte array.
address validatorAddress = popAddress(signature);

// Ensure signer has approved validator.
if (!allowedValidators[signerAddress][validatorAddress]) {
    return false;
}

magicValue = isValidValidatorSignature(
    validatorAddress,
    hash,
    signerAddress,
    signature
);
```

### PreSigned

Allows any address to sign a hash on-chain by calling the `preSign` method on the Exchange contract.

```
// Mapping of hash => signer => signed
mapping (bytes32 => mapping(address => bool)) public preSigned;

/// @dev Approves a hash on-chain using any valid signature type or `msg.sender`.
///      After presigning a hash, the preSign signature type will become valid for that hash and signer.
/// @param signerAddress Address that should have signed the given hash.
/// @param signature Proof that the hash has been signed by signer.
function preSign(
    bytes32 hash,
    address signerAddress,
    bytes signature
)
    external;
```

The hash can then be validated with only a `PreSigned` signature byte by checking the state of the `preSigned` mapping when a transaction is submitted.

```
isValid = preSigned[hash][signerAddress];
return isValid;
```

# Events

## Exchange events

### Fill

A `Fill` event is emitted when an order is filled.

```
event Fill(
    address indexed makerAddress,         // Address that created the order.
    address indexed feeRecipientAddress,  // Address that received fees.
    address takerAddress,                 // Address that filled the order.
    address senderAddress,                // Address that called the Exchange contract (msg.sender).
    uint256 makerAssetFilledAmount,       // Amount of makerAsset sold by maker and bought by taker.
    uint256 takerAssetFilledAmount,       // Amount of takerAsset sold by taker and bought by maker.
    uint256 makerFeePaid,                 // Amount of ZRX paid to feeRecipient by maker.
    uint256 takerFeePaid,                 // Amount of ZRX paid to feeRecipient by taker.
    bytes32 indexed orderHash,            // EIP712 hash of order (see LibOrder.getOrderHash).
    bytes makerAssetData,                 // Encoded data specific to makerAsset.
    bytes takerAssetData                  // Encoded data specific to takerAsset.
);
```

### Cancel

A `Cancel` event is emitted whenever an individual order is cancelled.

```
event Cancel(
    address indexed makerAddress,         // Address that created the order.
    address indexed feeRecipientAddress,  // Address that would have received fees if order was filled.
    address senderAddress,                // Address that called the Exchange contract (msg.sender).
    bytes32 indexed orderHash,            // EIP712 hash of order (see LibOrder.getOrderHash).
    bytes makerAssetData,                 // Encoded data specific to makerAsset.
    bytes takerAssetData                  // Encoded data specific to takerAsset.
);
```

### CancelUpTo

A `CancelUpTo` event is emitted whenever a [`cancelOrdersUpTo`](#cancelordersupto) call is successful.

```
event CancelUpTo(
    address indexed makerAddress,         // Orders cancelled must have been created by this address.
    address indexed senderAddress,        // Orders cancelled must have a `senderAddress` equal to this address.
    uint256 orderEpoch                    // Orders with specified makerAddress and senderAddress with a salt less than this value are considered cancelled.
);
```

### SignatureValidatorApproval

A `SignatureValidatorApproval` event is emitted whenever a [`Validator`](#validator) contract is approved or disapproved to verify signatures created by a signer via `setSignatureValidatorApproval`.

```
event SignatureValidatorApproval(
    address indexed signerAddress,     // Address that approves or disapproves a contract to verify signatures.
    address indexed validatorAddress,  // Address of signature validator contract.
    bool approved                      // Approval or disapproval of validator contract.
);
```

### AssetProxyRegistered

Whenever an [`AssetProxy`](#assetproxy) is registered the [`Exchange`](#exchange) contract, an `AssetProxyRegistered` is emitted.

```
event AssetProxyRegistered(
    uint8 id,               // Id of new registered AssetProxy.
    address assetProxy,     // Address of new registered AssetProxy.
);
```

## AssetProxy events

### AuthorizedAddressAdded

An `AuthorizedAddressAdded` event is emitted when a new address becomes authorized to call an [`AssetProxy`](#assetproxy) contract's transfer functions.

```
event AuthorizedAddressAdded(
    address indexed target,
    address indexed caller
);
```

### AuthorizedAddressRemoved

An `AuthorizedAddressRemoved` event is emitted when an address becomes unauthorized to call an [`AssetProxy`](#assetproxy) contract's transfer functions.

```
event AuthorizedAddressRemoved(
    address indexed target,
    address indexed caller
);
```

## AssetProxyOwner events

The following events must precede the execution of any function called by [`AssetProxyOwner`](#assetproxyowner) (with the exception of `removeAuthorizedAddressAtIndex`).

### Submission

A `Submission` event is emitted when a new transaction is submitted to the [`AssetProxyOwner`](#assetproxyowner).

```
event Submission(uint256 indexed transactionId);
```

### Confirmation

A `Confirmation` event is emitted when a transaction is confirmed by an individual owner of the [`AssetProxyOwner`](#assetproxyowner).

```
event Confirmation(
    address indexed sender,
    uint256 indexed transactionId
);
```

### ConfirmationTimeSet

A `ConfirmationTimeSet` event is emitted when a transaction has been fully confirmed. The 2 week timelock begins at this time, after which the transaction becomes executable.

```
event ConfirmationTimeSet(
    uint256 indexed transactionId,
    uint256 confirmationTime
);
```

# Types

## Order

```
struct Order {
    address makerAddress;           // Address that created the order.
    address takerAddress;           // Address that is allowed to fill the order. If set to 0, any address is allowed to fill the order.
    address feeRecipientAddress;    // Address that will receive fees when order is filled.
    address senderAddress;          // Address that is allowed to call Exchange contract methods that affect this order. If set to 0, any address is allowed to call these methods.
    uint256 makerAssetAmount;       // Amount of makerAsset being offered by maker. Must be greater than 0.
    uint256 takerAssetAmount;       // Amount of takerAsset being bid on by maker. Must be greater than 0.
    uint256 makerFee;               // Amount of ZRX paid to feeRecipient by maker when order is filled. If set to 0, no transfer of ZRX from maker to feeRecipient will be attempted.
    uint256 takerFee;               // Amount of ZRX paid to feeRecipient by taker when order is filled. If set to 0, no transfer of ZRX from taker to feeRecipient will be attempted.
    uint256 expirationTimeSeconds;  // Timestamp in seconds at which order expires.
    uint256 salt;                   // Arbitrary number to facilitate uniqueness of the order's hash.
    bytes makerAssetData;           // Encoded data that can be decoded by a specified proxy contract when transferring makerAsset. The last byte references the id of this proxy.
    bytes takerAssetData;           // Encoded data that can be decoded by a specified proxy contract when transferring takerAsset. The last byte references the id of this proxy.
}
```

## FillResults

Fill methods that return a value will return a FillResults instance if successful.

```
struct FillResults {
    uint256 makerAssetFilledAmount;  // Total amount of makerAsset(s) filled.
    uint256 takerAssetFilledAmount;  // Total amount of takerAsset(s) filled.
    uint256 makerFeePaid;            // Total amount of ZRX paid by maker(s) to feeRecipient(s).
    uint256 takerFeePaid;            // Total amount of ZRX paid by taker to feeRecipients(s).
}
```

## MatchedFillResults

The [`matchOrders`](#matchorders) method returns a MatchedFillResults instance if successful.

```
struct MatchedFillResults {
    FillResults left;                    // Amounts filled and fees paid of left order.
    FillResults right;                   // Amounts filled and fees paid of right order.
    uint256 leftMakerAssetSpreadAmount;  // Spread between price of left and right order, denominated in the left order's makerAsset, paid to taker.
}
```

## OrderInfo

The [`getOrderInfo`](#getorderinfo) method returns an `OrderInfo` instance.

```
struct OrderInfo {
    uint8 orderStatus;                    // Status that describes order's validity and fillability.
    bytes32 orderHash;                    // EIP712 hash of the order (see LibOrder.getOrderHash).
    uint256 orderTakerAssetFilledAmount;  // Amount of order that has already been filled.
}
```

# Standard relayer API

For a full specification of how orders are intended to be posted to and retrieved from relayers, see the [SRA v2 specification](https://github.com/0xProject/standard-relayer-api#sra-v2).

# Miscellaneous

## EIP712 usage

Hashes of orders and transactions are calculated according to the [EIP712 specification](https://github.com/ethereum/EIPs/pull/712/files).

The domain separator for the Exchange contract can be calculated with:

```
// EIP191 header for EIP712 prefix
string constant internal EIP191_HEADER = "\x19\x01";

// Hash of the EIP712 Domain Separator Schema
bytes32 constant internal EIP712_DOMAIN_SEPARATOR_SCHEMA_HASH = keccak256(abi.encodePacked(
    "EIP712Domain(",
    "string name,",
    "string version,",
    "address verifyingContract",
    ")"
));

bytes32 EIP712_DOMAIN_HASH = keccak256(abi.encodePacked(
    EIP712_DOMAIN_SEPARATOR_SCHEMA_HASH,
    keccak256(bytes("0x Protocol")),
    keccak256(bytes("2")),
    bytes32(address(this))
));
```

For more information about how this is used, see [hashing an order](#hashing-an-order) and [hashing a transaction](#hash-of-a-transaction).

## Optimizing calldata

Calldata is expensive. As per Appendix G of the [Ethereum Yellowpaper](#https://ethereum.github.io/yellowpaper/paper.pdf), every non-zero byte of calldata costs 68 gas, and every zero byte costs 4 gas. There are certain off-chain optimizations that can be made in order to maximize the amount of zeroes included in calldata.

### Filling remaining amounts

When an order is filled, it will attempt to fill the minimum of the amount submitted and the amount remaining. Therefore, if a user attempts to fill a very large amount such as `0xF000000000000000000000000000000000000000000000000000000000000000`, then the order will almost always be maximally filled while using minimal extra calldata.

### Filling orders that have already been partially filled

When filling an order, the signature is only validated the first time the order is filled. Because of this, signatures should _not_ be resubmitted after an order has already been partially filled. For a standard 65 byte ECDSA signature, this can save well over 4000 gas.

### Optimizing salt

When creating an order, a full 32 byte salt is generally unecessary to facilitate uniqueness of the order's hash. Using a salt value with as many leading zeroes as possible will increase gas efficiency. It is recommended to use a timestamp or incrementing nonce for the salt value, which will generally be small enough to optimize gas while also working well with [`cancelOrdersUpTo`](#cancelordersupto).

### Assuming order parameters

The [`matchOrders`](#matchorders), [`marketSellOrders`](#marketsellorders), [`marketSellOrdersNoThrow`](#marketsellordersnothrow), [`marketBuyOrders`](#marketbuyorders), and [`marketBuyOrdersNoThrow`](#marketbuyordersnothrow) functions all require that certain parameters of the later passed in orders match the same parameters of the first passed in order. Rather than checking equality, these functions all assume that the parameters are equal. This means users may pass in zero values for those parameters and the functions will still execute as if the values had been passed in as calldata.

### Vanity addresses

If frequently trading from a single address, it may make sense to generate a vanity address with as many zero bytes as possible.

## ecrecover usage

The `ecrecover` precompile available in Solidity expects `v` to always have a value of `27` or `28`. Some signers and clients assume that `v` will have a value of `0` or `1`, so it may be necessary to add `27` to `v` before submitting it to the `Exchange` contract.

## Reentrancy protection

The following functions within the `Exchange` contract contain a mutex that prevents them from called via [reentrancy](https://solidity.readthedocs.io/en/v0.4.24/security-considerations.html#re-entrancy):

- [`fillOrder`](#fillorder)
- [`fillOrKillOrder`](#fillorkillorder)
- [`batchFillOrders`](#batchfillorders)
- [`batchFillOrKillOrders`](#batchfillorkillorders)
- [`marketBuyOrders`](#marketbuyorders)
- [`marketSellOrders`](#marketsellorders)
- [`matchOrders`](#matchorders)
- [`cancelOrder`](#cancelorder)
- [`batchCancelOrders`](#batchcancelorders)
- [`cancelOrdersUpTo`](#cancelordersupto)
- [`setSignatureValidatorApproval`](#validator)

[`fillOrderNoThrow`](#fillordernothrow) and all of its variations do not explicitly have a mutex, but will fail gracefully if any reentrancy is attempted.

The mutex is implemented with the following `nonReentrant` modifier:

```
contract ReentrancyGuard {

    // Locked state of mutex
    bool private locked = false;

    /// @dev Functions with this modifer cannot be reentered. The mutex will be locked
    ///      before function execution and unlocked after.
    modifier nonReentrant() {
        // Ensure mutex is unlocked
        require(
            !locked,
            "REENTRANCY_ILLEGAL"
        );

        // Lock mutex before function call
        locked = true;

        // Perform function call
        _;

        // Unlock mutex after function call
        locked = false;
    }
}
```
