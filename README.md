# Notes from Solana Pathway - learn-web3-dapp

## Setup the project

1. To view solana version
   `solana --version`

2. To update Solana CLI
   `solana-install update`

3. At some point it may be useful to run a Test Validator. For example, if can't get airdrops because the Devnet faucet is not working. To run a local Test Validator, use this command in its own terminal tab or window :
   `solana-test-validator`

4. Use solana config set to target a particular cluster. After setting a cluster target, any future subcommands will send/receive information from that cluster. To target a running Test Validator with the Solana CLI:
   `solana config set --url https://localhost:8899`

5. You can see which cluster the Solana command-line tool (CLI) is currently targeting and the paths to your keypair and configuration file with the command :
   `solana config get`

## Connect to Solana

1. `@solana/web3.js` library is a convenient way to interface with the RPC API when building javscript applications. Under the hood it implements Solana's RPC methods and exposes them as javascript objects.

2. Use the `import {Connection} from '@solana/web3.js'` to connect to Solana.

   > See pages/api/solana/connect.ts for reference

## Create an account

1. Transactions on Solana happen between accounts. To create an account, a client generates a keypair which has a public key (or address, used to identify and lookup an account) and a secret key used to sign transactions.

   > See pages/api/solana/keypair.ts for reference

## Funct the account with SOL

1. To fund an account, we will do what is called an airdrop - some tokens will magically fall from the sky into our wallets. The cluster will provide us with some SOL so that we can test making transfers as well as view the transaction details on a block explorer. 1 SOL is equal to 1,000,000,000 lamports. Use the `fund` function to fund an account.

   > See pages/api/solana/fund.ts for reference

Token Info: With some protocols, different networks (testnet, mainnet, etc) have different token names. In the Solana world, the token is always called SOL, no matter what network (or cluster) you are on

## Get the balance

1. Check Balance to make sure there is sufficient SOL to perform a transfer.

   > See pages/api/solana/balance.ts for reference

## Transfer some SOL

1. To transfer some value to another account, we need to create and send a signed transaction to the cluster. When a transaction is submitted to the cluster, the Solana runtime will execute a program to process each of the instructions contained in the transaction, in order, and atomically. This means that if any of the instructions fail for any reason, the entire transaction will revert.

   > See pages/api/solana/transfer.ts for reference

## Deploy a program

1. A program is to Solana what a smart contract is to other protocols. Once a program has been deployed, any app can interact with it by sending a transaction containing the program instructions to a Solana cluster, which will pass it to the program to be run.

_Smart contract review_

There is a program folder at the app's root. It contains the Rust program contracts/solana/program/src/lib.rs and some configuration files to help us compile and deploy it. It's a simple program, all it does is increment a number every time it's called. Let’s dissect what each part does.

```
  use borsh::{BorshDeserialize, BorshSerialize};
  use solana_program::{
      account_info::{next_account_info, AccountInfo},
      entrypoint,
      entrypoint::ProgramResult,
      msg,
      program_error::ProgramError,
      pubkey::Pubkey,
  };

```

The `use` declarations are convenient shortcuts to other code. In this case, the serialize and de-serialize functions from the borsh crate. borsh stands for Binary Object Representation Serializer for Hashing.
A crate is a collection of source code which can be distributed and compiled together. Learn more about Cargo, Crates and basic project structure.

In this program we are going to `use` portions of the `solana_program` crate :

- The `account_info` (which returns the `next_account_info` as well as the struct for `AccountInfo`)
- The `entrypoint` macro and related `entrypoint::ProgramResult`
- The `msg` macro(for low-impact logging on the blockchain)
- `program_error::ProgramError` (which allows on-chain programs to implement program-specific error types and see them returned by the Solana runtime. A program-specific error may be any type that is represented as or serialized to a u32 integer)
- and the `pubkey::Pubkey` struct

Next ...

```
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct GreetingAccount {
    pub counter: u32,
}
```

Next we will use the `derive` macro to generate all the necessary boilerplate code to wrap our `GreetingAccount` struct. This happens behind the scenes during compile time with any `#[derive()]` macros. This boilerplate code that is inserted at compile time.

The struct declaration itself is simple, we are using the `pub` keyword to declare our struct publicly accessible, meaning other programs and functions can use it. The `struct` keyword is letting the compiler know that we are defining a struct named `GreetingAccount`, which has a single field: `counter` with a type of `u32` , an unsigned 32-bit integer. This means our counter cannot be larger than 4,294,967,295.

Next, we declare an entry point...

```
entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello World Rust program entrypoint");
    let accounts_iter = &mut accounts.iter();
    let account = next_account_info(accounts_iter)?;
```

The code above is the `process_instruction` function:

- The return value of the `process_instruction` entrypoint will be a `ProgramResult`.
  `Result` comes from the `std` crate and is used to express the possibility of error.
- For (debugging)[https://docs.solana.com/developing/on-chain-programs/debugging], we can print messages to the
  Program Log (with the msg!() macro)[https://docs.rs/solana-program/1.7.3/solana_program/macro.msg.html], rather than use `println!()` which would be prohibitive in terms of computational cost for the network.
- The `let` keyword in Rust binds a value to a variable. By looping through the `accounts` using an (iterator)[https://doc.rust-lang.org/book/ch13-02-iterators.html], `accounts_iter` is taking a (mutable)[https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html#mutable-references] reference of each value in `accounts`. Then `next_account_info(accounts_iter)?` will return the next `AccountInfo` or a `NotEnoughAccountKeys` error. Notice the `?` at the end, this is a (shortcut expression)[https://doc.rust-lang.org/std/result/#the-question-mark-operator-] in Rust for (error propagation)[https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html#propagating-errors].

Next ...

```
if account.owner != program_id {
  msg!("Greeted account does not have the correct program id");
  return Err(ProgramError::IncorrectProgramId);
}
```

The code above will perform a security check to see if the account owner has permission. If the `account.owner` public key does not equal the `program_id` we will return an error

Finally ...

```
let mut greeting_account = GreetingAccount::try_from_slice(&account.data.borrow())?;
greeting_account.counter += 1;
greeting_account.serialize(&mut &mut account.data.borrow_mut()[..])?;

msg!("Greeted {} time(s)!", greeting_account.counter);

Ok(())
```

The code above we "borrow" the existing account data, increase the value of `counter` by one and write it back to storage.

- The `GreetingAccount` struct has only one field - `counter`. To be able to modify it, we need to borrow the reference to `account.data` with the `&`(borrow operator)[https://doc.rust-lang.org/reference/expressions/operator-expr.html#borrow-operators].
- The `try_from_slice()` function from `BorshDeserialize` will mutably reference and deserialize the `account.data`.
- The `borrow()` function comes from the Rust core library, and exists to immutably borrow the wrapped value.

Taken together, this is saying that we will borrow the account data and pass it to a function that will deserialize it and return an error if one occurs. Recall that `?` is for error propagation.

Next, incrementing the value of `counter` by `1` is simple, using the addition assignment operator : `+=`.

With the `serialize()` function from `BorshSerialize`, the new `counter` value is sent back to Solana in the correct format. The mechanism by which this occurs is the (Write trait)[https://doc.rust-lang.org/std/io/trait.Write.html] from the `std::io` crate.

We can then show in the Program Log how many times the count has been incremented by using the `msg!()` macro.

_Setup the Solana CLI_

Windows is not compatible from this point on. To save a ton of headaches and issues. Install and use wsl2 for windows. Instruction are (here)[https://docs.microsoft.com/en-us/windows/wsl/install]

Install Rust and Solana CLI
Follow instruction under Linux and MacOS

1. (Install the latest Rust stable)[https://rustup.rs/]

2. (Install Solana CLI)[https://docs.solana.com/cli/install-solana-cli-tools]

3. The Linux Rust installer doesn't check for a compiler toolchain, but seems to assume that you've already got a C linker installed and you will need this to work. So last thing you might need to do is run `sudo apt install build-essential`

Now run this to enter the linux terminal in windows:

```
wsl
```

Now you can get started with following the rest of the instructions. Use the Linux and MacOS approach moving forward.

We need to configure the Solana cluster, create an account, request an airdrop and check that everything is functioning properly.

Set the CLI config URL to the devnet cluster:
`solana config set --url https://api.devnet.solana.com`

Next, we're going to generate a new keypair using the CLI. Run the following command in your Terminal:

```
mkdir solana-wallet
solana-keygen new --outfile solana-wallet/keypair.json
```

You will need SOL available in the account to deploy the program, so get an airdrop with:

On Linux or MacOS: `solana airdrop 1 $(solana-keygen pubkey solana-wallet/keypair.json)`

> IMPORTANT NOTE: If you're still running these instructions on a windows terminal and not on a wsl terminal you need to make sure the the `id.json` file is located in the right directory. For some weird reason, windows might create and adds your id.json in a User.config folder. Something similar to this `C:\Users\User.config\id.json`. To fix this and have the `id.json` file in the right directory you want it to be under a path similar to this `C:\Users\User\.config\solana\id.json` or `C:\Users\User\.config\id.json`. Do the following:
> Run this command first:

```
solana config set -k C:\Users\User\.config\solana\id.json

or

solana config set -k C:\Users\User\.config\id.json
```

Then run this command to see if the keypair path is configured in the right location:

```
solana config get
```

Then run this command to get the pubkey:

```
solana address
```

Then run this command to get the airdrop:

```
solana airdrop 1 <past your pubkey>
```

You should get something like this in return:

```
Requesting airdrop of 1 SOL

Signature: xxxxxxxxxxxxxxRandom Hashed Outputxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

1 SOL
```

Verify that everything is ok:
On Windows

```
solana config get
solana account <past your pubkey>
```

On Linux or MacOS

```
solana config get
solana account $(solana-keygen pubkey solana-wallet/keypair.json)
```

## Deploying a Solana program

1. When running the command `yarn run solana:build:program` it's going to call `cargo build-bpf --manifest-path=contracts/solana/program/Cargo.toml --bpf-out-dir=dist/solana/program`. `cargo build-bpf` does not work on windows so the solana build will not work. For the command to work you have to run it in wsl.

## Deploy a Program Challenge

1. After adding the solution to `pages/api/solana/deploy.ts` run `yarn dev` in a windows terminal. Not in wsl.

## Create Storage for the program

In solana programs are stateless. To store values we must use a seperate account from the account we created at the beginning of this tutorial. In this case we will create the **greeter account** to store the `count` info

> See pages/api/solana/greeter.ts for reference

## Get data from the program

Data is stored into an account as a buffer. To access the data, we'll have to first unpack this blob of data into a well defined structure.

> See pages/api/solana/getter.ts for reference

## Send data to the program

We'll need to modify the data stored in `greeter`. Doing so will change the state of the blockchain, so we'll have to create a transaction.

> See pages/api/solana/setter.ts for reference
