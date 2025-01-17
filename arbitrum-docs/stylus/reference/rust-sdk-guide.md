---
title: 'Stylus Rust SDK: Feature overview'
sidebar_label: 'Stylus Rust SDK: Feature overview'
description: 'An in-depth overview of the features provided by the Stylus Rust SDK'
author: rachel-bousfield
sme: rachel-bousfield
sidebar_position: 1
target_audience: Developers using the Stylus Rust SDK to write and deploy smart contracts.
---

import PublicPreviewBannerPartial from '../partials/_stylus-public-preview-banner-partial.md';

<PublicPreviewBannerPartial />

This document provides an in-depth overview of the features provided by the [Stylus Rust SDK](https://github.com/OffchainLabs/stylus-sdk-rs). For information about deploying Rust smart contracts, see the `cargo stylus` [CLI Tool](https://github.com/OffchainLabs/cargo-stylus). For a conceptual introduction to Stylus, see [Stylus: A Gentle Introduction](../stylus-gentle-introduction.md). To deploy your first Stylus smart contract using Rust, refer to the [Quickstart](../stylus-quickstart.md).

The Stylus Rust SDK is built on top of [Alloy](https://www.paradigm.xyz/2023/06/alloy), a collection of crates empowering the Rust Ethereum ecosystem. Because the SDK uses the same [Rust primitives for Ethereum types](https://docs.rs/alloy-primitives/latest/alloy_primitives/), Stylus is compatible with existing Rust libraries.

:::info

Many of the affordances use macros. Though this document details what each does, it may be helpful to use [`cargo expand`](https://crates.io/crates/cargo-expand) to see what they expand into if you’re doing advanced work in Rust.

:::

Additionally, the Stylus SDK supports `#[no_std]` for contracts that wish to opt out of the standard library. In fact, the entire SDK is available from `#[no_std]`, so no special feature flag is required. This can be helpful for reducing binary size, and may be preferable in pure-compute use cases like cryptography.

Most users will want to use the standard library, which is available since the Stylus VM supports `rustc`'s `wasm32-unknown-unknown` target triple. In the future we may add `wasm32-wasi` too, along with floating point and SIMD, which the Stylus VM does not yet support.

## Storage

Rust smart contracts may use state that persists across transactions. There’s two primary ways to define storage, depending on if you want to use Rust or Solidity definitions. Both are equivalent, and are up to the developer depending on their needs.

### `#[solidity_storage]`

The `#[solidity_storage]` macro allows a Rust struct to be used in persistent storage.

```rust
#[solidity_storage]
pub struct Contract {
    owner: StorageAddress,
    active: StorageBool,
    sub_struct: SubStruct,
}

#[solidity_storage]
pub struct SubStruct {
    // types implementing the `StorageType` trait.
}
```

Any type implementing the [`StorageType`](https://docs.rs/stylus-sdk/latest/stylus-sdk/trait.StorageType.html) trait may be used as a field, including other structs, which will implement the trait automatically when `#[solidity_storage]` is applied. You can even implement [`StorageType`](https://docs.rs/stylus-sdk/latest/stylus-sdk/trait.StorageType.html) yourself to define custom storage types. However, we’ve gone ahead and implemented the common ones.

| Type              | Info                                                                 |
| ----------------- | -------------------------------------------------------------------- |
| StorageBool       | Stores a bool                                                        |
| StorageAddress    | Stores an Alloy Address                                              |
| StorageUint       | Stores an Alloy Uint                                                 |
| StorageSigned     | Stores an Alloy Signed                                               |
| StorageFixedBytes | Stores an Alloy FixedBytes                                           |
| StorageBytes      | Stores a Solidity bytes                                              |
| StorageString     | Stores a Solidity string                                             |
| StorageVec        | Stores a vector of StorageType                                       |
| StorageMap        | Stores a mapping of StorageKey to StorageType                        |
| StorageArray      | Not yet implement, but will store a fixed-sized array of StorageType |

Every [Alloy primitive](https://docs.rs/alloy-primitives/latest/alloy_primitives/) has a corresponding `StorageType` implementation with the word `Storage` before it. This includes aliases, like `StorageU256` and `StorageB64`.

### `sol_storage!`

The types in `#[solidity_storage]` are laid out in the EVM state trie exactly as they are in [Solidity](https://docs.soliditylang.org/en/v0.8.20/abi-spec.html#basic-design). This means that the fields of a struct definition will map to the same storage slots as they would in EVM programming languages.

Because of this, it is often nice to define your types using Solidity syntax, which makes that guarantee easier to see. For example, the earlier Rust struct can re-written to:

```rust
sol_storage! {
    pub struct Contract {
        address owner;                      // becomes a StorageAddress
        bool active;                        // becomes a StorageBool
        SubStruct sub_struct,
    }

    pub struct SubStruct {
        // other solidity fields, such as
        mapping(address => uint) balances;  // becomes a StorageMap
        Delegate delegates[];               // becomes a StorageVec
    }
}
```

The above will expand to the equivalent definitions in Rust, each structure implementing the `StorageType` trait. Many contracts, like our example ERC 20, do exactly this.

Because the layout is identical to [Solidity’s](https://docs.soliditylang.org/en/v0.8.20/abi-spec.html#basic-design), existing Solidity smart contracts can upgrade to Rust without fear of storage slots not lining up. You simply copy-paste your type definitions.

:::tip
Existing Solidity smart contracts can upgrade to Rust if they use proxy patterns
:::

Consequently, the order of fields will affect the JSON ABIs produced that explorers and tooling might use. Most developers won’t need to worry about this though and can freely order their types when working on a Rust contract from scratch.

### Reading and writing storage

You can access storage types via getters and setters. For example, the `Contract` struct from earlier might access its `owner` address as follows.

```rust
impl Contract {
    /// Gets the owner from storage.
    pub fn owner(&self) -> Result<Address, Vec<u8>> {
        Ok(self.owner.get())
    }

    /// Updates the owner in storage
    pub fn set_owner(&mut self, new_owner: Address) -> Result<(), Vec<u8>> {
        if msg::sender() == self.owner()? {  // we'll discuss msg::sender later
            self.owner.set(new_owner);
        }
        Ok(())
    }
}
```

In Solidity, one has to be very careful about storage access patterns. Getting or setting the same value twice doubles costs, leading developers to avoid storage access at all costs. By contrast, the Stylus SDK employs an optimal storage-caching policy that avoids the underlying `SLOAD` or `SSTORE` operations.

:::tip
Stylus uses storage caching, so multiple accesses of the same variable is virtually free.
:::

However it must be said that storage is ultimately more expensive than memory. So if a value doesn’t need to be stored in state, you probably shouldn’t do it.

### Collections

Collections like `StorageVec` and `StorageMap` are dynamic and have methods like `push`, `insert`, `replace`, and similar.

```rust
impl SubStruct {
    pub fn add_delegate(&mut self, delegate: Delegate) -> Result<(), Vec<u8>> {
        self.delegates.push(delegate);
    }

    pub fn track_balance(&mut self, address: Address) -> Result<U256, Vec<u8>> {
        self.balances.insert(address, address.balance());
    }
}
```

You may notice that some methods return types like `StorageGuard` and `StorageGuardMut`. This allows us to leverage the Rust borrow checker for storage mistakes, just like it does for memory. Here’s an example that will fail to compile.

```rust
fn mistake(vec: &mut StorageVec<StorageU64>) -> U64 {
    let value = vec.setter(0);
    let alias = vec.setter(0);
    value.set(32.into());
    alias.set(48.into());
    value.get() // uh, oh. what value should be returned?
}
```

Under the hood, `vec.setter()` returns a `StorageGuardMut` instead of a `&mut StorageU64`. Because the guard is bound to a `&mut StorageVec` lifetime, `value` and `alias` cannot be alive simultaneously. This causes the Rust compiler to reject the above code, saving you from entire classes of storage aliasing errors.

In this way the Stylus SDK safeguards storage access the same way Rust ensures memory safety. It should never be possible to alias Storage without `unsafe` Rust.

### `SimpleStorageType`

You may run into scenarios where a collection’s methods like `push` and `insert` aren’t available. This is because only primitives, which implement a special trait called `SimpleStorageType`, can be added to a collection by value. For nested collections, one instead uses the equivalent `grow` and `setter`.

```rust
fn nested_vec(vec: &mut StorageVec<StorageVec<StorageU8>>) {
    let mut inner = vec.grow();  // adds a new element accessible via `inner`
    inner.push(0.into());        // inner is a guard to a StorageVec<StorageU8>
}

fn nested_map(map: &mut StorageMap<u32, StorageVec<U8>>) {
    let mut slot = map.setter(0);
    slot.push(0);
}
```

### `Erase` and `#[derive(Erase)]`

Some `StorageType` values implement `Erase`, which provides an `erase()` method for clearing state. We’ve implemented `Erase` for all primitives, and for vectors of primitives, but not maps. This is because a solidity `mapping` does not provide iteration, and so it’s generally impossible to know which slots to set to zero.

Structs may also be `Erase` if all of the fields are. `#[derive(Erase)]` lets you do this automatically.

```rust
sol_storage! {
    #[derive(Erase)]
    pub struct Contract {
        address owner;              // can erase primitive
        uint256[] hashes;           // can erase vector of primitive
    }

    pub struct NotErase {
        mapping(address => uint) balances; // can't erase a map
        mapping(uint => uint)[] roots;     // can't erase vector of maps
    }
}
```

You can also implement `Erase` manually if desired. Note that the reason we care about `Erase` at all is that you get storage refunds when clearing state, lowering fees. There’s also minor implications for patterns using `unsafe` Rust.

### The storage cache

The Stylus SDK employs an optimal storage-caching policy that avoids the underlying `SLOAD` or `SSTORE` operations needed to get and set state. For the vast majority of use cases, this happens in the background and requires no input from the user.

However, developers working with `unsafe` Rust implementing their own custom `StorageType` collections, the `StorageCache` type enables direct control over this data structure. Included are `unsafe` methods for manipulating the cache directly, as well as for bypassing it altogether.

### Immutables and `PhantomData`

So that generics are possible in `sol_interface!`, `core::marker::PhantomData` implements `StorageType` and takes up zero space, ensuring that it won’t cause storage slots to change. This can be useful when writing libraries.

```rust
pub trait Erc20Params {
    const NAME: &'static str;
    const SYMBOL: &'static str;
    const DECIMALS: u8;
}

sol_storage! {
    pub struct Erc20<T> {
        mapping(address => uint256) balances;
        PhantomData<T> phantom;
    }
}
```

The above allows consumers of `Erc20` to choose immutable constants via specialization. See our WETH sample contract for a full example of this feature.

### Future storage work

The Stylus SDK is currently in alpha and will improve in the coming versions. Something you may notice is that storage access patterns are often a bit verbose. This will change soon when we implement `DerefMut` for most types.

Another area for improvement we expect to land very soon is support for fixed arrays of `StorageType`. There’s nothing conceptually different about them, but we just haven’t finished implementing them. There’s other improvements we intend to make to structs too.

## Methods

Just as with storage, Stylus SDK methods are Solidity ABI equivalent. This means that contracts written in different programming languages are fully interoperable. As detailed in this section, you can even automatically export your Rust contract as a Solidity interface so that others can add it to their Solidity projects.

:::tip
Stylus programs compose regardless of programming language. For example, existing Solidity DEXs can list Rust ERC-20s without modification.
:::

### `#[external]`

This macro makes methods “external” so that other contracts can call them by implementing the `Router` trait.

```rust
#[external]
impl Contract {
    // our owner method is now callable by other contracts
    pub fn owner(&self) -> Result<Address, Vec<u8>> {
        Ok(self.owner.get())
    }
}

impl Contract {
    // our set_owner method is not
    pub fn set_owner(&mut self, new_owner: Address) -> Result<(), Vec<u8>> {
        ...
    }
}
```

Note that, currently, all external methods must return a `Result` with the error type `Vec<u8>`. We intend to change this very soon. In the current model, `Vec<u8>` becomes the program’s revert data, which we intend to both make optional and richly typed.

### `#[payable]`

As in Solidity, methods may accept ETH as call value.

```rust
#[external]
impl Contract {
    #[payable]
    pub fn credit(&mut self) -> Result<(), Vec<u8> {
        self.erc20.add_balance(msg::sender(), msg::value())
    }
}
```

In the above, `msg::value` is the amount of ETH passed to the contract in wei, which may be used to pay for something depending on the contract’s business logic. Note that you have to annotate the method with `#[payable]`, or else calls to it will revert. This is required as a safety measure since it prevents vulnerabilities based on covertly updating contract balances.

### `#[pure]`, `#[view]`, and `#[write]`

For aesthetics, these additional purity attributes exist to clarify that a method is `pure`, `view`, or `write`. They aren’t necessary though, since the `#[external]` macro can figure purity out for you based on the types of the arguments.

For example, if a method includes an `&self`, it’s at least `view`. If you’d prefer it be `write`, applying `#[write]` will make it so. Note however that the reverse is not allowed. An `&mut self` method cannot be made `#[view]`, since it might mutate state.

### `#[entrypoint]`

This macro allows you to define the entrypoint, which is where Stylus execution begins. Without it, the contract will fail to pass `cargo stylus check`. Most commonly, the macro is used to annotate the top level storage struct.

```rust
sol_storage! {
    #[entrypoint]
    pub struct Contract {
        ...
    }

    // only one entrypoint is allowed
    pub struct SubStruct {
        ...
    }
}
```

The above will make the external methods of `Contract` the first to consider during invocation. In a later section we’ll discuss inheritance, which will allow the `#[external]` methods of other types to be invoked as well.

### Bytes-in, bytes-out programming

A less common usage of `#[entrypoint]` is for low-level, bytes-in bytes-out programming. When applied to a free-standing function, a different way of writing smart contracts becomes possible, where the Rust SDK’s macros and storage types are entirely optional.

```rust
#[entrypoint]
fn entrypoint(calldata: Vec<u8>) -> ArbResult {
    // bytes-in, bytes-out programming
}
```

### Reentrancy

If a contract calls another that then calls the first, it is said to be reentrant. By default, all Stylus programs revert when this happened. However, you can opt out of this behavior by customizing your entrypoint.

```rust
#[entrypoint(allow_reentrancy = true)]
```

This is dangerous, and should be done only after careful review — ideally by 3rd party auditors. Numerous exploits and hacks have in Web3 are attributable to developers misusing or not fully understanding reentrant patterns.

If enabled, the Stylus SDK will flush the storage cache in between reentrant calls, persisting values to state that might be used by inner calls. Note that preventing storage invalidation is only part of the battle in the fight against exploits. You can tell if a call is reentrant via `msg::reentrant`, and condition your business logic accordingly.

### `TopLevelStorage`

The `#[entrypoint]` macro will automatically implement the `TopLevelStorage` trait for the annotated `struct`. The single type implementing `TopLevelStorage` is special in that mutable access to it represents mutable access to the entire program’s state. This idea will become important when discussing calls to other programs in later sections.

### Inheritance, `#[inherit]`, and `#[borrow]`.

Composition in Rust follows that of Solidity. Types that implement `Router`, the trait that `#[external]` provides, can be connected via inheritance.

```rust
#[external]
#[inherit(Erc20)]
impl Token {
    pub fn mint(&mut self, amount: U256) -> Result<(), Vec<u8>> {
        ...
    }
}

#[external]
impl Erc20 {
    pub fn balance_of() -> Result<U256> {
        ...
    }
}
```

Because `Token` inherits `Erc20` in the above, if `Token` has the `#[entrypoint]`, calls to the contract will first check if the requested method exists within `Token`. If a matching function is not found, it will then try the `Erc20`. Only after trying everything `Token` inherits will the call revert.

Note that because methods are checked in that order, if both implement the same method, the one in `Token` will override the one in `Erc20`, which won’t be callable. This allows for patterns where the developer imports a crate implementing a standard, like the ERC 20, and then adds or overrides just the methods they want to without modifying the imported `Erc20` type.

Inheritance can also be chained. `#[inherit(Erc20, Erc721)]` will inherit both `Erc20` and `Erc721`, checking for methods in that order. `Erc20` and `Erc721` may also inherit other types themselves. Method resolution finds the first matching method by [Depth First Search](https://en.wikipedia.org/wiki/Depth-first_search).

Note that for the above to work, `Token` must implement `Borrow<Erc20>`. You can implement this yourself, but for simplicity, `#[solidity_storage]` and `sol_storage!` provide a `#[borrow]` annotation.

```rust
sol_storage! {
    #[entrypoint]
    pub struct Token {
        #[borrow]
        Erc20 erc20;
        ...
    }

    pub struct Erc20 {
        ...
    }
}
```

In the future we plan to simplify the SDK so that `Borrow` isn’t needed and so that `Router` composition is more configurable. The motivation for this becomes clearer in complex cases of multi-level inheritance, which we intend to improve.

## Exporting a Solidity interface

Recall that Stylus contracts are fully interoperable across all languages, including Solidity. The Stylus SDK provides tools for exporting a Solidity interface for your contract so that others can call it. This is usually done with the `cargo stylus` [CLI tool](https://github.com/OffchainLabs/cargo-stylus#exporting-solidity-abis), but we’ll detail how to do it manually here.

The SDK does this automatically for you via a feature flag called `export-abi` that causes the `#[external]` and `#[entrypoint]` macros to generate a `main` function that prints the Solidity ABI to the console.

```rust
cargo run --features export-abi --target <triple>
```

Note that because the above actually generates a `main` function that you need to run, the target can’t be `wasm32-unknown-unknown` like normal. Instead you’ll need to pass in your target triple, which `cargo stylus` figures out for you. This `main` function is also why the following commonly appears in the `[main.rs](http://main.rs)` file of Stylus contracts.

```rust
#![cfg_attr(not(feature = "export-abi"), no_main)]
```

Here’s an example output. Observe that the method names change from Rust’s `snake_case` to Solidity’s `camelCase`. For compatibility reasons, onchain method selectors are always `camelCase`. We’ll provide the ability to customize selectors very soon. Note too that you can use argument names like `address` without fear. The SDK will prepend an `_` when necessary.

```solidity
interface Erc20 {
    function name() external pure returns (string memory);

    function balanceOf(address _address) external view returns (uint256);
}

interface Weth is Erc20 {
    function mint() external payable;

    function burn(uint256 amount) external;
}
```

## Calls

:::caution UNDER CONSTRUCTION

This section is currently under construction, and will be updated soon.

If you're waiting for this content to be completed, click the `Request an update` button at the top of this page to let us know!

:::

### `sol_interface!`

_Coming soon!_

### Call contexts

_Coming soon!_

### Calls with inheritance

_Coming soon!_

### `transfer_eth`

_Coming soon!_

### `RawCall` and `unsafe` calls

Occasionally, an untyped call to another contract is necessary. `RawCall` lets you configure an `unsafe` call by calling optional configuration methods. This is similar to how one would configure opening a `File` in Rust.

```rust
let data = RawCall::new_delegate()   // configure a delegate call
    .gas(2100)                       // supply 2100 gas
    .limit_return_data(0, 32)        // only read the first 32 bytes back
    .flush_storage_cache()           // flush the storage cache before the call
    .call(contract, calldata)?;      // do the call
```

Note that the `call` method is `unsafe`. This is due to reentrancy, and the fact that the call does not require clearing the storage cache.

## `RawDeploy` and `unsafe` deployments

Right now the only way to deploy a contract from inside Rust is to use `RawDeploy`, similar to `RawCall`. As with `RawCall`, this mechanism is inherently unsafe due to reentrancy concerns, and requires manual management of the `StorageCache`.

Note that the EVM allows init code to make calls to other contracts, which provides a vector for reentrancy. This means that this technique may enable storage aliasing if used in the middle of a storage reference's lifetime and if reentrancy is allowed.

When configured with a `salt`, `RawDeploy` will use `CREATE2` instead of the default `CREATE`, facilitating address determinism.

## Events

Emitting Solidity-style events is supported out-of-the-box with the Rust SDK. They can be defined in Solidity syntax using Alloy’s `sol!` macro, and then used as input arguments to `evm::log`. The function accepts any type that implements Alloy’s `SolEvent` trait.

```rust
sol! {
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

fn foo() {
   ...
   evm::log(Transfer {
      from: Address::ZERO,
      to: address,
      value,
   });
}
```

The SDK also exposes a low-level, `evm::raw_log` that takes in raw bytes and topics:

```rust
/// Emits an evm log from combined topics and data. The topics come first.
fn emit_log(bytes: &[u8], topics: usize)
```

## EVM affordances

The SDK contains several modules for interacting with the EVM, which can be imported like so.

```rust
use stylus_sdk::{block, contract, crypto, evm, msg, tx};

let callvalue = msg::value();
```

| Rust SDK Module | Description                                           |
| --------------- | ----------------------------------------------------- |
| block           | block info for the number, timestamp, etc.            |
| contract        | contract info, such as its address, balance           |
| crypto          | VM accelerated cryptography                           |
| evm             | ink / memory access functions                         |
| msg             | sender, value, and reentrancy detection               |
| tx              | gas price, ink price, origin, and other tx-level info |
