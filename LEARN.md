# Creating NFTs using NEAR on Rust
Near supports multiple programming languages. At the time of this writing the most recommended programming language for creating contracts is Rust. Near supports Rust, AssemblyScript that is very similar to Typescript and Solidity that is used in Ethereum too. 

However the support is best documented and maintained in Rust making it a good choice for writing code on Near inspite of it’s steep learning curve. However, many other blockchains like Solana also use Rust. So building some muscle on writing contracts in Rust is a good idea for developers looking to break into crypto.

NEAR contracts written in Rust are smaller, cheaper and more secure. 

We require you to have gone through the basics of NEAR track on [https://questbook.app](https://questbook.app) before we dive into Rust based programming of smart contracts. Writing code on Assembly Script is definitely easier for the uninitialized.

In this quest, we’ll start building an NFT. We’ll introduce you to the tools that are required to build and some basic programming. We will be writing code for the actual NFT itself in the next quest.
## Setting up Rust
We first need to install Rust. 

Open up a terminal and punch in the following installation command from the official rust servers.

```

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

```

Once installed, we’ll need to install the WebAssembly toolchain. This will enable us to compile Rust into a WebAssembly file that can be run on the blockchain. 

```

rustup target add wasm32-unknown-unknown

```

Once this is successfully installed, you are ready to start building.
## Setting up the project
First up lets create a directory for where we’ll write our first NEAR contract project. 

```

mkdir nearnft

cd nearnft

```

Initialize this project by using Rust’s cargo command

```

cargo init --lib

```

This initializes the project as a library. That way it can be used NEAR blockchain. 

You’ll notice Rust creates a project directory with a `Cargo.toml` file that is the configurations for this project and a `src/lib.rs` rust file where we’ll be writing our contract. 

Add the following to your `Cargo.toml`

```

\[lib\]

crate-type = \["cdylib", "rlib"\]

\[dependencies\]

near-sdk = "3.1.0"

```

This basically tells Rust to create a dynamic library and asks it to install the dependency or near-sdk.

Now let’s open up `src/lib.rs` and replace the contents with the following imports. 

```

use near_sdk::borsh::{self, BorshDeserialize, BorshSerialize};

use near_sdk::{env, near_bindgen};

near_sdk::setup_alloc!();

```

We will also need to import the serializing libraries so that we are able to convert our variables into data that will be permanently stored on the blockchain. 

With this we’re good to start writing our core logic.
## Defining an NFT
For this quest we will create a super simple NFT data structure, and improve it to adhere to standards over the next few quests. 

Let us first define our NFT datastructure. It is a very simple mapping between the NFT’s token ID and the owner of that NFT.

```

pub struct NftOwners {

  owners: UnorderedMap;

}

```

We’ll assume that the token Id and the owner’s address are both going to be strings. 

We’ll use the UnorderedMap which is nothing but a dictionary but recognized by Near’s blockchain.
## Making the data blockchain friendly
Now that we’ve created our datastructure, we need to make it storable on the blockchain. 

We’re already using `UnorderedMap`. This data type is imported from the near-sdk. 

Add the import line to the top of your file

```

use near_sdk::collections::UnorderedMap;

```

And lastly, make the data serializable using the following decorators

```

\#\[near_bindgen\]

\#\[derive(BorshDeserialize, BorshSerialize)\]

pub struct NftOwnership {

  owners: UnorderedMap

}

```

We’ll take a moment to understand why we need to do this in Near. Why could we not directly create a variable `owners` why did we have to put it inside a `struct`?

The answer to this lies in the fact that, on near there are accounts. 

Each account can consist of the following things. 

- The Account ID
- Balance in Near Tokens
- At most 1 contract
- Storage

When we will be deploying a contract, we will be creating an account and putting the contract inside that account. 

Every contract being a part of an account also has the native support to handle and store money.

The storage is independent of the contract space. So, we need to make sure that the contract stores the information in a way that is consistent and usable even outside of the contract. Remember all the data on the blockchain is public information.

So, to make it storable in the storage part of the account, we need to create a struct that is serializable. The struct is stored in a serializable way. So when we want to alter the `owners` variable the near-runtime will pull the data from the storage space, deserialize it, make the modifications, serialize it and store it back in the storage.
## The logic
First up we need to create a default for our NftOwners struct

```

impl Default for NftOwners {

  fn default() -> Self {

    Self { 

        owners: UnorderedMap::new(b“o”.to_vec())

    }

  }

} 

```

Defaults are required for structs in Rust. 

You can think of this as the constructor for the ‘class’

When we create a new `UnorderedMap` we also need to provide an identifier `”o”` which will identify this map on the storage space. 

```

\#\[near_bindgen\]

impl NftOwners {

  pub fn setOwner(&mut self, tokenId: String, accountId: String){

    self.owners.insert(&tokenId, &accountId);

  }

pub fn getOwner(&self, tokenId: String) -> Option {

    return self.owners.get(&tokenId);

  }

}

```

Here we’ve created the two functions to set and get the owners of a said NFT.

You’ll notice some Rust specific syntax here if you are new to the language. 

This impl is like an implementation of the functions of a class. 

The first parameter of the functions is always the &self object. In some programming languages this is called the `this` pointer. 

You’ll notice that the `setOwner` uses `&mut self` whereas `getOwner` uses `&self`. 

This is because if you have to make any alterations to a variable in Rust you must use the `mut` keyword, standing for mutable. In `setOwner` we will be updating the dictionary, so we need that to be mutable. 

The return type of `getOwner` is `Option` meaning it can be a `String` or a `null`.

Great, now we’ve written our first contract! Lets compile and deploy this!
## Deploy the contract
First compile the contract using 

```

env 'RUSTFLAGS=-C link-arg=-s' 

cargo build --target wasm32-unknown-unknown --release

```

Basically telling rust to compile it using the wasm toolchain. 

This will create a wasm file in `targets/wasm32-unknown-unknown/release`

To deploy this `wasm` file, 

```

near dev-deploy --wasmFile target/wasm32-unknown-unknown/release/nearnft.wasm 

```

This will deploy the contract to the test network

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/b0908d28-b9e9-4222-8c68-f645effc41d5.jpg)

You should see the link to the deployed contract on the console. This is called a deployment transaction. There are a few other kinds of transactions like transfer (of tokens) and function calls.  The account ID here is the contract name. Remember that an account can have utmost one contract. So, to identify a contract it is enough to identify it by the account ID. Copy this and keep it handy, we’ll refer to this as `CONTRACT_NAME` when we need to use it. 

To be able to make function call transactions you need an account. 

On your commandline enter 

```

near login

```

After a successful login, the public and private keys will be stored in your machine at `~/near-credentials/testnet/yourname.testnet`

Now that you have deployed the contract and have an account, you can interact with it using 

```

near call CONTRACT_NAME setOwner '{"tokenId": "firstNFT", "owner" : "madhavan.testnet"}' --accountId yourname.testnet

```

Notice the way to pass parameters is in the form of a JSON. 

Lastly,  you should also be able to call the getOwner function right after this!
## What Next?
Great stuff - you’ve created a contract using rust and deployed it. 

In the next quest, we’ll be looking at how we can make the contract more robust and adhering to the standards laidout by the NEAR community :)