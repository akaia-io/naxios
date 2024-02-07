<div align="center">
   <img src="https://raw.githubusercontent.com/wpdas/naxios/HEAD/md/naxios-logo.png" height="70px" /></a><br>
</div>

<p align="center">Promise based NEAR Contract and NEAR Wallet client for browser. This was designed to facilitate the React integration with NEAR Blockchain and avoid the huge boilerplate of setting up a wallet and contract.</p>

<p align="center">
    <a href="https://wpdas.gitbook.io/naxios/"><b>Documentation</b></a>
</p>

## Table of Contents

- [Features](#features)
- [Installing](#installing)
- [How to Use](#how-to-use)
- [React Hooks](#react-hooks)
- [Utils](#utils)
- [Contributing](#contributing)

## Features

- Simplified Wallet and Contract integration
- Supports the [Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) API
- Easy to create a Contract Interface
- Wallet connection modal to be easily used
- Automatic transforms for [JSON](https://www.json.org/json-en.html) data
- Client side events to tell when the api is ready
- Helpful react hooks
- Cache System for contract `view`

## Installing

Using npm:

```bash
npm install @wpdas/naxios @near-wallet-selector/modal-ui@8.9.1
```

Using yarn:

```bash
yarn add @wpdas/naxios @near-wallet-selector/modal-ui@8.9.1
```

## How to Use

### Preparing it

Import the NEAR Wallet Selector styles. The app needs it to render the Wallet Selector correctly.

```ts
import '@near-wallet-selector/modal-ui/styles.css'
```

### Using It

It's super easy to get a Wallet and/or Contract API in place all at once. Take a look:

```ts
// web3Api.ts
import naxios from '@wpdas/naxios'

const naxiosApi = new naxios({
  contractId: CONTRACT_ID,
  network: 'testnet', // or mainnet, localnet
})

// (optional)
const onContractInitHandler = () => {
  console.log('Contract is ready!')
}
const onWalletInitHandler = () => {
  console.log('Wallet is ready!')
}

/**
 * NEAR Contract API
 */
export const contractApi = naxiosInstance.contractApi(onContractInitHandler)

/**
 * NEAR Wallet API
 */
export const walletApi = naxiosInstance.walletApi(onWalletInitHandler)
```

You can also invoke a new instance anywhere, anytime with a new configuration if you wish:

```ts
// Invoking a Contract API
import { getContractApi } from '@wpdas/naxios'

// New contract instance (This also supports cache)
const contract = await getContractApi({
  contractId: ANOTHER_CONTRACT_ID,
  network: 'testnet',
})

const response = await contract.view('get_users', {
  args: { limit: 5 },
})
```

```ts
// Invoking a Wallet API
import { getWalletApi } from '@wpdas/naxios'

// New wallet instance
const walletApi = await getWalletApi({
  contractId: ANOTHER_CONTRACT_ID,
  network: 'testnet',
})

// Open up the Signin Wallet Modal
walletApi.signInModal()
```

### Cache System

There are two kinds of cache systems to be used. They are `Memory Cache` and `Storage Cache`.

`Memory Cache`: will be cleared when the app refreshes, as its data lives in memory only. <br/>
`Storage Cache`: The data will remain even when the browser tab is refreshed. Data is persisted using Local Storage.

When instantiating a cache, you need to provide the `expirationTime` (in seconds). This is used to know when the cache should be returned instead of making a real contract call. When the cache expires, a real call to the contract is made. Each contract's method has its own time of expiration.

```ts
// web3Api.ts with cache
import naxios, { StorageCache } from '@wpdas/naxios'

const naxiosInstance = new naxios({
  contractId: CONTRACT_ID,
  network: 'testnet',
  cache: new StorageCache({ expirationTime: 60 }),
})

export const contractApi = naxiosInstance.contractApi()
```

Then, to use cached `view`, you can just pass the configuration object saying you want to use cached data.

```ts
import { contractApi } from './web3Api'

const args: {}
const config: { useCache: true }

contractApi.view('get_greeting', args, config).then((response) => console.log(response))
```

#### Contract API Reference

- `view`: Make a read-only call to retrieve information from the network. It has the following parameters:
  - `method`: Contract's method name.
  - `props?`: an optional parameter with `args` for the contract's method.
  - `config?`: currently, this has only the `useCache` prop. When useCache is true, this is going to use non-expired cached data instead of calling the contract's method.
- `call`: Call a method that changes the contract's state. This is payable. It has the following parameters:
  - `method`: Contract's method name
  - `props?`: an optional parameter with `args` for the contract's method, `gas`, `deposit` to be attached and `callbackUrl` if you want to take the user to a specific page after a transaction succeeds.
- `callMultiple`: Call multiple methods that change the contract's state. This is payable and has the following parameters:
  - `transactionsList`: A list of Transaction props. You can use `buildTransaction(...)` to help you out
  - `callbackUrl?`: A page to take the user to after all the transactions succeed.

#### Wallet API Reference

- `accounts`: Signed-in Accounts.
- `accountId`: Main/first signed-in account ID in the accounts list.
- `contractId`: Contract ID.
- `initNear`: (This is called automatically. You don't need to call it!) Initializes a connection to the NEAR blockchain.
- `network`: Current network (`testnet`, `mainnet` or `localnet`).
- `signInModal`: Open up the Signin Wallet Modal.
- `wallet`: Wallet instance.
- `walletSelector`: WalletSelector instance.

### Contract View

Using a `view` method is free.

```ts
import { contractApi } from './web3Api'

contractApi.view('get_greeting').then((response) => console.log(response))
// Hi
```

### Contract Call

You need to pay for every request you make for a `call` method. This is going to change data and store it within the blockchain.

```ts
import { contractApi } from './web3Api'

// [payable]
contractApi.call('set_greeting', { greeting: 'Hello my dear!' }).then(() => console.log('Done!'))
```

### Contract Multiple Calls at Once

As well as the `call`, you will need to pay for every request you make. This is going to change data and store it within the blockchain.

```ts
import { buildTransaction } from '@wpdas/naxios'
import { contractApi } from './web3Api'

// Using the default instance's contract
const transactionA = buildTransaction('set_greeting', { args: { greeting: 'Hello my dear!' } })
const transactionB = buildTransaction('set_age', { args: { age: 22 } })
// Using diff contract
const transactionC = buildTransaction('update_state', {
  receiverId: 'my-state-contract.testnet',
  args: { allowed: true },
})

// Optional
const callbackUrl = 'https://my-page.com/callback-success'

// [payable]
contractApi.callMultiple([transactionA, transactionB, transactionC], callbackUrl).then(() => console.log('Done!'))
```

### Contract Interface

It's a good practice to create a Contract Interface while building your app, so that, everyone knows what to input and what to get at the end.

```ts
// contract-interface.ts
import { contractApi } from './web3Api'

// Get greeting request
type EmptyInput = {}
type GetGreetingResponse = string
export const get_greeting = () => contractApi.view<EmptyInput, GetGreetingResponse>('get_greeting')

// [payable]
// Set greeting request
type SetGreetingInput = { greeting: string }
type SetGreetingResponse = string // current greeting
export const set_greeting = (args: SetGreetingInput) =>
  contractApi.call<typeof args, SetGreetingResponse>('set_greeting', { args })
```

Then, you can just call it over your app like:

```ts
import { useState, useEffect, useCallback } from 'react'
import { get_greeting, set_greeting } from './contract-interface'

const App = () => {
  const [greeting, setGreeting] = useState('')

  // Loads the last stored greeting
  useEffect(() => {
    ;(async () => {
      const storedGreeting = await get_greeting()
      setGreeting(storedGreeting)
    })()
  }, [])

  // Persist a new greeting message
  const persistNewGreetingHandler = useCallback(async () => {
    await set_greeting({ greeting: 'Hello my dear!!!' })
    console.log('Done!')
  }, [])

  return (
    <>
      <button onClick={persistNewGreetingHandler}>Save new greeting</button>
    </>
  )
}
```

### Open Up the Sign-in Wallet Selector Modal

You can open up the NEAR Wallet Selector modal by calling `signInModal()`:

```ts
import { walletApi } from './web3Api'

walletApi.signInModal()
```

<div align="center">
   <img src="https://3798793431-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fcudid4AoeKizKC6M6ros%2Fuploads%2FjD0nWoQ61lkQeEZwflHy%2FScreenshot%202024-01-11%20at%2009.15.59.png?alt=media&token=90d7edf4-fc7a-4879-916e-f0d2f9cace3f" height="100%" maxHeight="500px" /></a><br>
</div>

### Customizing the Wallets Options for NEAR Wallet Selector

By default, naxios only uses **@near-wallet-selector/my-near-wallet** as a means of connecting the wallet. However, you can add other wallet selectors as follows:

```sh
npm install @near-wallet-selector/ledger @near-wallet-selector/my-near-wallet
```

Then, you can start naxius as follows:

```ts
import naxios from '@wpdas/naxios'
import { setupMyNearWallet } from '@near-wallet-selector/my-near-wallet'
import { setupLedger } from '@near-wallet-selector/ledger'

const naxiosApi = new naxios({
  contractId: CONTRACT_ID,
  network: 'testnet', // or mainnet, localnet
  walletSelectorModules: [setupMyNearWallet(), setupLedger()],
})
```

Find out all the NEAR wallet selectors here: [**NEAR Wallet Selector**](https://github.com/near/wallet-selector)

## React Hooks

#### `useContract`

The `useContract` hook initializes a connection to the NEAR Blockchain and provides access to the contractApi instance.

```tsx
const contract = useContract({ contractId: CONTRACT_ID, network: 'testnet' })

useEffect(() => {
  if (contract.ready) {
    contract.view('get_greeting').then((response) => console.log(response)) // Hi
  }
}, [contract])
```

#### API

- `ready`: boolean indicating whether the contract API is ready.
- `view`: Make a read-only call to retrieve information from the network. It has the following parameters:
  - `method`: Contract's method name.
  - `props?`: an optional parameter with `args` for the contract's method.
  - `config?`: currently, this has only the `useCache` prop. When useCache is true, this is going to use non-expired cached data instead of calling the contract's method.
- `call`: Call a method that changes the contract's state. This is payable. It has the following parameters:
  - `method`: Contract's method name
  - `props?`: an optional parameter with `args` for the contract's method, `gas`, `deposit` to be attached and `callbackUrl` if you want to take the user to a specific page after a transaction succeeds.
- `callMultiple`: Call multiple methods that change the contract's state. This is payable and has the following parameters:
  - `transactionsList`: A list of Transaction props. You can use `buildTransaction(...)` to help you out
  - `callbackUrl?`: A page to take the user to after all the transactions succeed.

<!-- To add a separator line -->

##

<!-- To add a separator line -->

#### `useWallet`

The `useWallet` hook initializes a connection to the NEAR Blockchain and provides access to the walletApi instance.

```tsx
const wallet = useWallet({ contractId: CONTRACT_ID, network: 'testnet' })

useEffect(() => {
  if (wallet.ready) {
    console.log(wallet.walletApi?.accounts)
    // [{accountId: 'user.testnet', publicKey: 'ed25519:aaaaaaaa'}, {...}]
  }
}, [wallet])
```

#### API

- `ready`: boolean indicating whether the wallet API is ready.
- `walletApi`: Wallet API. See its [API Reference here](#wallet-api-reference).

## Utils

#### `buildTransaction`

The `buildTransaction` method is useful when you need to build a contract's Transaction body, mainly when you want to make [**multiple contract calls**](#contract-multiple-calls-at-once).

[**See reference here.**](#contract-multiple-calls-at-once)

#### `validateNearAddress`

This is used to check if an address is a valid NEAR address.

```ts
import { validateNearAddress } from '@wpdas/naxios'

console.log(validateNearAddress('fake.near')) // true
console.log(validateNearAddress('fake.nears')) // false
console.log(validateNearAddress('fake.testnet')) // true
console.log(validateNearAddress('fake')) // false
```

#### `calculateDepositByDataSize`

Calculate required deposit for data being stored. (~0.00001N per byte) with a bit extra for buffer

```ts
import { calculateDepositByDataSize } from '@wpdas/naxios'

const myData = { age: 22, name: 'user name' }
console.log(calculateDepositByDataSize(myData)) // 0.00087 Near (not yocto NEAR)
```

## Contributing

Feel free to open issues or pull requests. For major changes, please open an issue first to discuss what you would like to change.
