# Creating simple NFTs using NEAR on Rust
Near supports multiple programming languages. At the time of this writing the most recommended programming language for creating contracts is Rust. 
The support is best documented and maintained in Rust making it a good choice for writing code on Near inspite of it’s steep learning curve. However, many other blockchains like Solana also use Rust. So building some muscle on writing contracts in Rust is a good idea for developers looking to break into crypto.

NEAR contracts written in Rust are smaller, cheaper and more secure. 

We require you to have gone through the basics of NEAR track on [https://questbook.app](https://questbook.app) before we dive into Rust based programming of smart contracts. Writing code on Assembly Script is definitely easier for the uninitialized.

In this quest, we’ll start building a simple NFT. We’ll introduce you to the tools that are required to build and some basic programming. We will be writing code for the actual NFT itself in the next quest.

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

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
near-sdk = "3.1.0"

```

This basically tells Rust to create a dynamic library and asks it to install the dependency or near-sdk.

Now let’s open up `src/lib.rs` and replace the contents with the following imports. 

```

use near_sdk::borsh::{self, BorshDeserialize, BorshSerialize};
use near_sdk::{near_bindgen, AccountId};
near_sdk::setup_alloc!();

```

We will also need to import the serializing libraries so that we are able to convert our variables into data that will be permanently stored on the blockchain. 

With this we’re good to start writing our core logic.

## Defining an NFT
For this quest we will create a super simple NFT data structure, and improve it to adhere to standards over the next few quests. 

Let us first define our NFT datastructure. It is a very simple mapping between the NFT’s token ID and the owner of that NFT.

```

pub struct NftOwners {
    owners: UnorderedMap<String, AccountId>,
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

And lastly, make the data serializable using the following decorators:

```

#[near_bindgen]
#[derive(BorshDeserialize, BorshSerialize)]

pub struct NftOwners {

 owners: UnorderedMap<String, AccountId>,

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
            owners: UnorderedMap::new(b"o"),
        }
    }
}

```

Defaults are required for structs in Rust. 

You can think of this as the constructor for the ‘class’

When we create a new `UnorderedMap` we also need to provide an identifier `”o”` which will identify this map on the storage space. 

```

#[near_bindgen]
impl NftOwners {
    pub fn set_owner(&mut self, token_id: String, account_id: AccountId) {
        self.owners.insert(&token_id, &account_id);
    }

    pub fn get_owner(&self, token_id: String) -> AccountId {
        match self.owners.get(&token_id) {
            Some(owner) => owner,
            None => "No owner found".to_string(),
        }
    }
}
```

Here we’ve created the two functions to set and get the owners of a said NFT.

You’ll notice some Rust specific syntax here if you are new to the language. 

This impl is like an implementation of the functions of a class. 

The first parameter of the functions is always the &self object. In some programming languages this is called the `this` pointer. 

You’ll notice that the `setOwner` uses `&mut self` whereas `getOwner` uses `&self`. 

This is because if you have to make any alterations to a variable in Rust you must use the `mut` keyword, standing for mutable. In `setOwner` we will be updating the dictionary, so we need that to be mutable. 

Great, now we’ve written our first contract! Lets compile and deploy this!

## Compiling, deploying, and calling the contract:
Now go to your project root, create a scripts directory, and cd into it. 
Here we will write a bunch of .sh files to do the job.
Create a build.sh, and add the follwing to it:

```

!#/bin/bash
env 'RUSTFLAGS=-C link-arg=-s'
cargo build --target wasm32-unknown-unknown --release
cp ../target/wasm32-unknown-unknown/release/nearnft.wasm ../result/result.wasm

```

This will compile your contract and copy the resulting .wasm to a result directory, were you can view it easier. Create the result directory before running this bash script. Now run ``` sh build.sh ```. You will have to wait a bit, this command will install some dependencies first.

Now let's deploy. Create a deploy.sh and add this:

```

near dev-deploy --wasmFile ../result/result.wasm

```
Run it, grab this ``` dev-***-*** ```, this is the most important thing now. 
Of course, export a $CONTRACT with your contract's ID so we can use it when calling functions, it is just more convenient.

``` export CONTRACT=dev-***-*** ```

Now run 
``` near login ```
So you can interact with the contract. Give access from your browser and let's move on.
Create a use-contract.sh, add the following:

```

near call $CONTRACT set_owner '{"token_id": "firstNFT", "account_id" : "you.testnet"}' --accountId "you.testnet"
near view $CONTRACT get_owner '{"token_id": "firstNFT"}' 

```

replace you.testnet with your account ID. 
Can you guess what that view call will return? Yep, your NEAR account ID (you.testnet). Run it to view the result.
Cool? Now let's see how to test this thing.

## Testing the contract:
Add these to lib.rs:

```

#[cfg(test)]
mod tests {
    use super::*;
    use near_sdk::MockedBlockchain;
    use near_sdk::{testing_env, VMContext};
    fn get_context(predecessor_account_id: String, storage_usage: u64) -> VMContext {
        VMContext {
            current_account_id: "alice.testnet".to_string(),
            signer_account_id: "jane.testnet".to_string(),
            signer_account_pk: vec![0, 1, 2],
            predecessor_account_id,
            input: vec![],
            block_index: 0,
            block_timestamp: 0,
            account_balance: 0,
            account_locked_balance: 0,
            storage_usage,
            attached_deposit: 0,
            prepaid_gas: 10u64.pow(18),
            random_seed: vec![0, 1, 2],
            is_view: false,
            output_data_receivers: vec![],
            epoch_height: 19,
        }
    }

    #[test]
    fn set_owner() {
        let context = get_context("you.testnet".to_string(), 0);
        testing_env!(context);
        let mut contract = NftOwners::default();
        let my_token = "my_token".to_string();
        let owner_account_id = "you.testnet".to_string();
        contract.set_owner(my_token.clone(), owner_account_id.clone());
        let owner_of_nft = contract.get_owner(my_token);
        assert_eq!(owner_of_nft, owner_account_id);
    }
}

```
Looks scary? Don't worry. See that getContext() function? This is just a boilerplate that sets up a mock blockchain for you to test on. The real work is in set_owner(). You can see that we just created a token and asserted that the contract stored the right owner.
Run this to trigget the test:

```

cargo test -- --nocapture

```

You should get something like this:

```

running 1 test
test tests::set_owner ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s


```
And that is about it!

## What Next?
Great stuff - you’ve created a contract using rust, deployed it, and tested it. 

In the next quest, we’ll be looking at how we can make the contract more robust and adhering to the standards laidout by the NEAR community :)
