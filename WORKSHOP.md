# RISC Zero Workshop: On-chain Privacy Preserving Proofs using Composition & Bonsai

This workshop is an exploration of a admittedly very dense project to reveal various killer features that [RISC Zero enables](https://dev.risczero.com/api/use-cases):

1. Local ("client side") _privacy preserving_ Zero Knowledge Proofs (not _just_ Succinct ones!)
1. [Proof Composition](https://dev.risczero.com/api/zkvm/composition) that uses recursion to efficiently verify proofs with another proof
1. Remote ("server side") proving with service providers like [Bonsai](https://dev.risczero.com/litepaper)
1. Solidity verifiers that make these proofs cost-effective [EVM Coprocessors](https://dev.risczero.com/api/blockchain-integration/bonsai-on-eth) to use On-chain in your contracts.
1. An Orchestration service that is the glue making this whole thing "just work" 😀

## Get ready...

1. Setup your env: https://dev.risczero.com/api/zkvm/install

   ```sh
   curl -L https://risczero.com/install | bash
   rzup
   # then follow printed instructions to install toolchain
   ```

1. Clone the base [Foundry Template](https://github.com/risc0/risc0-foundry-template/)

   ```sh
   # Dir name suggesting what we are up to 😉
   git clone --depth=1 https://github.com/risc0/risc0-foundry-template.git risc0-client-server-proof-composition
   ```

1. Request [Bonsai API Key](https://www.bonsai.xyz/) (Fill out the form)
   > The workshop host should have access to approve instantly & share the key

## [Project Structure](https://github.com/risc0/risc0-foundry-template?tab=readme-ov-file#project-structure)

The key files to read through, with ⭐ on the ones we will modify the most:

```sh
.
├── Cargo.toml                      // Configuration for Cargo and Rust
├── foundry.toml                    // Configuration for Foundry
├── apps
│   ├── Cargo.toml
│   └── src
│       └── lib.rs                  // Utility functions
│       └── bin
│           └── publisher.rs        // *Orchestration service* to create proofs and ship them to an EVM contract
├── contracts
│   ├── EvenNumber.sol              // ⭐ Basic example contract for you to hack on
│   └── ImageID.sol                 // *Generated* contract with the image ID for your zkVM program
├── methods
│   ├── Cargo.toml
│   ├── guest
│   │   ├── Cargo.toml
│   │   └── src
│   │       └── bin                 // ⭐ You can add additional guest programs to this folder
│   │           └── is_even.rs      // ⭐ Example guest program for checking if a number is even
│   └── src
│       └── lib.rs                  // Compiled image IDs and tests for your guest programs
└── tests
    ├── EvenNumber.t.sol            // ⭐ Tests for the basic example contract
    └── Elf.sol                     // *Generated* contract with paths the guest program ELF files.
```

## Set... OK Lezzzz GO!

1. Follow along with the [README.md](./README.md) to ensure tests pass
1. Follow along with the [deployment-guide.md](./deployment-guide.md) to ensure tests pass here too
   > NOTE: as mentioned, producing the [Groth16 SNARK proofs][Groth16] ON YOUR MACHINE (not bonsai) for this test requires running on an x86 machine with [Docker] installed.
   > Apple silicon is currently unsupported for local proving, you can find out more info in the relevant issues [here](https://github.com/risc0/risc0/issues/1520) and [here](https://github.com/risc0/risc0/issues/1749).

All tests should pass before you move on 🙏

## Welcome our new `guest`

Let's introduce a new `guest` program that proves we know a secret number $x$ and computed in $x^e \mod{n}$ :

`methods/guest/src/bin/power-modulus.rs`

```rs
//! Stolen from https://github.com/risc0/risc0/tree/main/examples/composition
//! This example is analogous to verifiable encryption with RSA.

use risc0_zkvm::{guest::env, serde};

fn main() {
    // n and e are the public modulus and exponent respectively.
    // x value that will be kept private.
    let (n, e, x): (u64, u64, u64) = env::read();

    // Commit n, e, and x^e mod n.
    env::commit(&(n, e, pow_mod(x, e, n)));
}

/// Compute x^e (mod n)
pub fn pow_mod(x: u64, mut e: u64, n: u64) -> u64 {
    let mut x = x as u128;
    let n = n as u128;
    let mut z = 1u128;

    // Apply a simple implementation of exponentiation by squaring
    // https://en.wikipedia.org/wiki/Exponentiation_by_squaring
    while e > 0 {
        if e % 2 == 1 {
            z = (z * x) % n
        }
        e >>= 1;
        x = (x * x) % n;
    }
    return z as u64;
}
```

Link it up to our build process:

`methods/guest/Cargo.toml`

```toml
# Add anywhere you like
[[bin]]
name = "power-modulus"
path = "src/bin/power_modulus.rs"
```

Build our `guest` executables:

```sh
cargo build --release
```

> Note: Our `methods/build.rs` produces [ELF](https://dev.risczero.com/terminology#elf-binary) and an [Image ID](https://dev.risczero.com/terminology#image-id) in `risc0-client-server-proof-composition/target/release/build/methods-<some random string>/out/methods.rs` for each `guest` program.
> We `include!(concat!(env!("OUT_DIR"), "/methods.rs"));` to get those `pub const` items in scope to use in proving, this is for both local and remote requests: we build the executable locally in both cases.

## 🤝 Guests keep their composure

We want to prove that our calculated exponent is even on-chain without revealing to _anyone_ what the original $x$ is.
Thus we need to [compose](https://dev.risczero.com/terminology#composition) `power_modulus.rs` with `is_even.rs`.
To do this we need to **orchestrate** the generation of the proofs and ultimately send a transaction that is verified on-chain - we can do that by modifying [`apps/src/bin/publisher.rs`](apps/src/bin/publisher.rs) (using the [composition example app](https://github.com/risc0/risc0/blob/main/examples/composition/src/main.rs) as reference) we want to:

1. _Locally_ (client side) produce a [receipt](https://dev.risczero.com/terminology#receipt) of our modulo math _locally_ with [`risc0_zkvm::LocalProver`](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/struct.LocalProver.html)
1. _Remotely_ (server side) request a [groth16 receipt](https://dev.risczero.com/terminology#groth16-receipt) that includes the local proof to fulfil it's [assumption](https://dev.risczero.com/terminology#assumption) when proving our local proof's result is even with [`risc0_zkvm::BonsaiProver`](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/struct.BonsaiProver.html)
1. Submit a transaction on chain that verifies this whole **blockchain coprocessing** workflow happened _correctly_, triggering a contract do a thing! 🤑
