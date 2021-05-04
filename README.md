<p align="center">
  <img src="https://app.1inch.io/assets/images/logo.svg" width="200" alt="1inch network" />
</p>

# Utils for limit orders protocol

This is the package of utilities for working with the `1inch limit orders protocol`

[LimitOrderBuilder](./src/limit-order.builder.ts) - to create a limit order  
[LimitOrderPredicateBuilder](./src/limit-order-predicate.builder.ts) - to create a predicates for limit order  
[LimitOrderProtocolFacade](./src/limit-order-protocol.facade.ts) - to interact with the protocol on the blockchain

---

## Test coverage

| Statements                                                                    | Branches                                                                    | Functions                                                                  | Lines                                                                    |
| ----------------------------------------------------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| ![Statements](https://img.shields.io/badge/Coverage-98.94%25-brightgreen.svg) | ![Branches](https://img.shields.io/badge/Coverage-96.83%25-brightgreen.svg) | ![Functions](https://img.shields.io/badge/Coverage-100%25-brightgreen.svg) | ![Lines](https://img.shields.io/badge/Coverage-98.94%25-brightgreen.svg) |

## Installation

### Node

```
npm install web3
```

### Yarn

```
yarn install web3
```

## Protocol addresses

-   Ethereum mainnet: `0x94a68df7e81b90a9007db9db7ffb3e6a2f1e6c1b`
-   BSC mainnet: `0x0e6b8845f6a316f92efbaf30af21ff9e78f0008f`

---

## Contents

0. [Quick start](#Quick-start)
1. [Create a limit order](#Create-a-limit-order)
2. [Check a limit order remaining](#Check-a-limit-order-remaining)
3. [Nonce](#Nonce)
4. [Validate a limit order](#Validate-a-limit-order)
5. [Create a predicate for limit order](#Create-a-predicate-for-limit-order)
6. [Fill a limit order](#Fill-a-limit-order)
7. [Cancel a limit order](#Cancel-a-limit-order)
8. [Cancel all limit orders](#Cancel-all-limit-orders)

## Quick start

```typescript
import {
    LimitOrderBuilder,
    LimitOrderProtocolFacade,
} from '@1inch/limit-order-protocol';

const contractAddress = '0xabc...';
const walletAddress = '0xzxy...';
const chainId = 1;

const web3 = new Web3('...');
// You can create and use a custom provider connector (for example: ethers)
const connector = new Web3ProviderConnector(web3);

const limitOrderBuilder = new LimitOrderBuilder(
    contractAddress,
    chainId,
    connector
);

const limitOrderProtocolFacade = new LimitOrderProtocolFacade(
    contractAddress,
    connector
);

// Create a limit order and signature
const limitOrder = limitOrderBuilder.buildOrder({
    makerAssetAddress: '0xbb4cdb9cbd36b01bd1cbaebf2de08d9173bc095c',
    takerAssetAddress: '0x111111111117dc0aa78b770fa6a738034120c302',
    makerAddress: walletAddress,
    makerAmount: '100',
    takerAmount: '200',
    predicate: '0x0',
    permit: '0x0',
    interaction: '0x0',
});
const limitOrderTypedData = limitOrderBuilder.buildOrderTypedData(limitOrder);
const limitOrderSignature = limitOrderBuilder.buildOrderSignature(
    walletAddress,
    limitOrderTypedData
);

// Fill the limit order
const callData = limitOrderProtocolFacade.fillOrder(
    limitOrder,
    limitOrderSignature,
    '100',
    '0'
);

sendTransaction({
    from: walletAddress,
    gas: 210_000, // Set your gas limit
    gasPrice: 40000, // Set your gas price
    to: contractAddress,
    data: callData,
});
```

**Note:** you can use any implementation for the provider.  
Just implement `ProviderConnector` interface:

```typescript
class MyProviderConnector implements ProviderConnector {
    //...
}
```

## Create a limit order

`LimitOrderBuilder.buildOrder()`

Parameters for creating a limit order:

-   `makerAssetAddress` - address of maker token
-   `takerAssetAddress` - address of taker token
-   `makerAddress` - address of maker
-   `takerAddress` - address of taker. Default: `0x0000000000000000000000000000000000000000`
-   `makerAmount` - amount of maker token, in wei units
-   `takerAmount` - amount of taker token, in wei units
-   `predicate` - predicate call data. Default: `0x`
-   `permit` - permit call data. Default: `0x`
-   `interaction` - interaction call data. Default: `0x`

### Example:

```typescript
import {LimitOrderBuilder} from '@1inch/limit-order-protocol';

const limitOrderBuilder = new LimitOrderBuilder();
// ...

const limitOrder = limitOrderBuilder.buildOrder({
    makerAssetAddress: '0xbb4cdb9cbd36b01bd1cbaebf2de08d9173bc095c',
    takerAssetAddress: '0x111111111117dc0aa78b770fa6a738034120c302',
    makerAddress: '0xfb3c7ebccccAA12B5A884d612393969Adddddddd',
    makerAmount: '100',
    takerAmount: '200',
    predicate: '0x0',
    permit: '0x0',
    interaction: '0x0',
});
const limitOrderTypedData = limitOrderBuilder.buildOrderTypedData(limitOrder);
const limitOrderSignature = limitOrderBuilder.buildOrderSignature(
    walletAddress,
    limitOrderTypedData
);
const limitOrderHash = limitOrderBuilder.buildOrderHash(limitOrderTypedData);
```

## Check a limit order remaining

`LimitOrderProtocolFacade.remaining()`

By default, a limit order is created unfilled.  
Until the first fill the `remaining` method will throw error `LOP: Unknown order`.  
After the first fill, the method will return remaining amount.

> Note: a limit order can be partially filled

### Example:

```typescript
import {
    LimitOrderProtocolFacade,
    LimitOrderHash,
} from '@1inch/limit-order-protocol';
import {BigNumber} from 'ethers/utils';

const orderMakerAmount = '400000000000'; // initial amount of the limit order
const orderHash: LimitOrderHash = '0xabc...';
const contractAddress = '0xabc...';

const connector = new Web3ProviderConnector(new Web3('...'));
const limitOrderProtocolFacade = new limitOrderProtocolFacade(
    contractAddress,
    connector
);

const remaining = await getRemaining(orderHash);

async function getRemaining(orderHash: string): string {
    try {
        const remaining: BigNumber = limitOrderProtocolFacade.remaining(
            orderHash
        );

        return remaining.toString();
    } catch (error) {
        const errorMessage = typeof error === 'string' ? error : error.message;

        if (errorMessage.includes('LOP: Unknown order')) {
            return orderMakerAmount;
        }

        throw error;
    }
}
```

## Nonce

`LimitOrderProtocolFacade.nonces()`

**Nonce** - this is the so-called `series` of limit orders.  
The nonce is useful when you need to create a bunch of limit orders with the ability to cancel them all later.

### Example:

```typescript
import {
    LimitOrderProtocolFacade,
    LimitOrderPredicateBuilder
} from '@1inch/limit-order-protocol';

const walletAddress = '0xhhh...';
const contractAddress = '0xabc...';
const chainId = 1;

const connector = new Web3ProviderConnector(new Web3('...'));
const limitOrderProtocolFacade = new limitOrderProtocolFacade(contractAddress, connector);
const limitOrderPredicateBuilder = new LimitOrderPredicateBuilder(
    limitOrderProtocolFacade
);
const limitOrderBuilder = new LimitOrderBuilder(
    contractAddress,
    chainId,
    connector
);

// Get the current nonce
const nonce = await limitOrderProtocolFacade.nonces(contractAddress);

// Create a limit order with nonceEquals predicate
const predicate = limitOrderPredicateBuilder.nonceEquals(walletAddress, nonce);
const limitOrder = limitOrderBuilder.buildOrder({
    ...,
    predicate
});

// Cancel all orders by advance nonce
const cancelAllOrdersCallData = limitOrderProtocolFacade.advanceNonce();
sendTransaction({
    from: walletAddress,
    gas: 210_000, // Set your gas limit
    gasPrice: 40000, // Set your gas price
    to: contractAddress,
    data: cancelAllOrdersCallData,
});
```

## Validate a limit order

`LimitOrderProtocolFacade.simulateTransferFroms()`

There is the possibility to check limit orders validity.  
For example: you can check that a limit order is valid by predicates.

> **Under the hood:**  
> On a `simulateTransferFroms()` call, the contract returns the string like `TRANSFERS_SUCCESSFUL_01101`  
> If that string contains at least one `0` symbol, then a limit order is invalid, otherwise - valid

### Example:

```typescript
import {LimitOrderProtocolFacade, LimitOrder} from '@1inch/limit-order-protocol';

const contractAddress = '0xabc...';
const order: LimitOrder = {...};

const connector = new Web3ProviderConnector(new Web3('...'));
const limitOrderProtocolFacade = new limitOrderProtocolFacade(contractAddress, connector);

const addresses = [contractAddress];
const callDatas = [order.predicate];

try {
    const result: boolean = await limitOrderProtocolFacade.simulateTransferFroms(addresses, callDatas);

    console.log('Order validity: ', result);
} catch (error) {
    console.error(error);
}
```

## Create a predicate for limit order

`LimitOrderPredicateBuilder`

A limit order can contain one or more predicates which indicate the logic of its validity.  
**There are two types of a predicate operators:**

### Conditional operators:

-   `and` - combine several predicates, return `true` when all predicates are valid
-   `or` - combine several predicates, return `true` when the one of predicates is valid

### Comparative operators:

> All comparative operators have three arguments:  
> (**value**: string, **address**: string, **callData**: string)

> **How the operators works:**  
> On an operator call, the contract execute the `callData` for the `address` and compare _**a result**_ with the `value`

-   `eq` - _**a result**_ must be equal to the `value`
-   `lt` - _**a result**_ must be less than the `value`
-   `gt` - _**a result**_ must be greater than the `value`

### Built-in operators:

> `nonceEquals(makerAddress: string, makerNonce: number)`

The predicate checks that the `makerNonce` is equal to the nonce of `makerAddress`

---

> `timestampBelow(timestamp: number)`

The predicate checks that `timestamp` is greater than the current time

### Example:

```typescript
import {
    LimitOrderProtocolFacade,
    LimitOrderPredicateBuilder,
    LimitOrderPredicateCallData
} from '@1inch/limit-order-protocol';

const makerAddress = '0xabc...';
const tokenAddress = '0xsss...';
const balanceOfCallData = '0xccc...';

const limitOrderProtocolFacade = new LimitOrderProtocolFacade(...);

const limitOrderPredicateBuilder = new LimitOrderPredicateBuilder(
    limitOrderProtocolFacade
);

const {or, and, timestampBelow, nonceEquals, gt, lt, eq} = predicateBuilder;

const simplePredicate: LimitOrderPredicateCallData = and(
    timestampBelow(Math.round(Date.now() / 1000) + 60_000), // a limit order is valid only for 1 minute
    nonceEquals(makerAddress, 4) // a limit order is valid until the nonce of makerAddress is equal to 4
);

const complexPredicate: LimitOrderPredicateCallData = or(
    and(
        timestampBelow(Math.round(Date.now() / 1000) + 60_000),
        nonceEquals(makerAddress, 4),
        gt('10', tokenAddress, balanceOfCallData)
    ),
    or(
        timestampBelow(5444440000),
        lt('20', tokenAddress, balanceOfCallData)
    ),
    eq('30', tokenAddress, balanceOfCallData)
);
```

## Fill a limit order

`LimitOrderProtocolFacade.fillOrder()`

Parameters for order filling:

-   `order: LimitOrder`
-   `signature: LimitOrderSignature`
-   `makerAmount: string`
-   `takerAmount: string`

> Note: to fill a limit order, only one of the amounts must be specified  
> The second one must be set to `0`

### Example

```typescript
import {
    LimitOrderProtocolFacade,
    LimitOrder,
    LimitOrderSignature
} from '@1inch/limit-order-protocol';

const walletAddress = '0xhhh...';
const contractAddress = '0xabc...';

const order: LimitOrder = {...};
const signature: LimitOrderSignature = '...';

const makerAmount = '400000000';
const takerAmount = '0';

const connector = new Web3ProviderConnector(new Web3('...'));
const limitOrderProtocolFacade = new limitOrderProtocolFacade(contractAddress, connector);

const callData = limitOrderProtocolFacade.fillOrder(
    order,
    signature,
    makerAmount,
    takerAmount
);

sendTransaction({
    from: walletAddress,
    gas: 210_000, // Set your gas limit
    gasPrice: 40000, // Set your gas price
    to: contractAddress,
    data: callData,
});
```

## Cancel a limit order

`LimitOrderProtocolFacade.cancelOrder()`

### Example:

```typescript
import {
    LimitOrderProtocolFacade,
    LimitOrder
} from '@1inch/limit-order-protocol';

const walletAddress = '0xhhh...';
const contractAddress = '0xabc...';

const order: LimitOrder = {...};

const connector = new Web3ProviderConnector(new Web3('...'));
const limitOrderProtocolFacade = new limitOrderProtocolFacade(contractAddress, connector);

const callData = limitOrderProtocolFacade.cancelOrder(order);

sendTransaction({
    from: walletAddress,
    gas: 210_000, // Set your gas limit
    gasPrice: 40000, // Set your gas price
    to: contractAddress,
    data: callData,
});
```

## Cancel all limit orders

`LimitOrderProtocolFacade.advanceNonce()`

### First of all, read about [Nonce](#nonce)

`advanceNonce()` increments the nonce and all limit orders with a predicate to the previous nonce value become invalid

> **Warning!**  
> The approach only works when all orders have the `nonceEquals` predicate

### Example:

```typescript
import {
    LimitOrderProtocolFacade,
    LimitOrder,
} from '@1inch/limit-order-protocol';

const walletAddress = '0xhhh...';
const contractAddress = '0xabc...';

const connector = new Web3ProviderConnector(new Web3('...'));
const limitOrderProtocolFacade = new limitOrderProtocolFacade(
    contractAddress,
    connector
);

const callData = limitOrderProtocolFacade.advanceNonce();

sendTransaction({
    from: walletAddress,
    gas: 210_000, // Set your gas limit
    gasPrice: 40000, // Set your gas price
    to: contractAddress,
    data: callData,
});
```
