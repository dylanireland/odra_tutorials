# Election Contract with Odra

## Introduction

This tutorial will guide you through the creation of a voting smart contract using Odra. The contract will be built assuming the following principles:

* The deployer can specify candidates and a final voting time in the constructor.
* The final voting time is denominated in block height.
* The deployer cannot modify the candidates or end time after deployment.
* Any account, besides the deployer, may make one vote for any candidate they please.

The contract could be extended to allow for modifications to candidates, end time, and voting capabilities, but this tutorial avoids these functionalities in the interest of simplicity.

## Preparation

Initialize a new Odra project:

```bash
cargo odra new --name election -t blank
```

*Note: The `-t blank` flag will create the contract as a blank template.*

Open *election/src/lib.rs* in an editor.

Begin the contract by importing the Odra datatypes `Var`, `Mapping`, and `Address`, as well as the `Vec` class from `alloc::vec`:

```rust
use odra::{Var, Mapping, Address};
use alloc::vec::Vec;
```

`Var` is Odra's most basic data structure that supports reading and writing to global state using built-in methods. `Mapping` has similar features, but is for key-value pairs. An `Address`, on the other hand, is an enum representing either an `AccountHash` or `ContractPackageHash` from [casper_types](https://docs.rs/casper-types/latest/casper_types/).
`Vec` is a well-known Rust struct representing a contiguous growable array type; we can wrap `Vec` with `Var`, to be able to save and read `Vec`s to and from global state.

## Interfacing

The module definition for the Election contract can be defined next, which needs to contain all of the objects to be stored by the contract:

```rust
#[odra::module]
pub struct Election {
    end_block: Var<u64>,
    candidates: Var<Vec<String>>,
    candidate_votes: Mapping<String, u32>,
    voters: Mapping<Address, bool>
}
```

In this implementation, the objects reference the following:

* `end_block`

  The final block in which voting is permitted

* `candidates`

  A vector of candidate names. For this example it will be assumed that there are no duplicates, however this should be validated in a production environment.

* `candidate_votes`

  A dictionary of candidate to vote count; a running tally of votes accrued by each candidate.

* `voters`

  A dictionary of accounts that have voted. If the boolean `true` is present under an `Address`, that account has already voted and may not vote again.

## User Errors

Preparing custom user errors is recommended and can provide insight when debugging or experiencing issues. Odra has an enum `OdraError` that can be derived to create custom errors. For this example, define the following errors:

```rust
#[derive(OdraError)]
pub enum Error {
    VotingEnded = 0,
    VoterAlreadyVoted = 1,
    ProbablyBecauseTheValueIsEmpty = 2
}
```

## Contract Implementation

To begin writing the smart contract's functionality, implement the `Election` module, marking the implementation with the `#[odra::module]` attribute:

```rust
#[odra::module]
impl Election {
	 
}
```

### Constructor

Define the constructor, which must be named `init`:

```rust
pub fn init(&mut self, end_block: u64, candidates: Vec<String>) {
        
}
```

The `&mut self` parameter allows us to access and mutate `self`, while the `end_block` and `candidates` parameters allow us to pass in values upon deployment.

Within the constructor, start by setting the end block to that passed in as an argument:

```rust
self.end_block.set(end_block);
```

Then iterate over `candidates` and use each candidate string to prepare the `candidate_votes` dictionary:

```rust
for candidate in candidates.iter() {
	candidate_votes.set(&candidate, 0u32);
}
```

### Vote Entrypoint

Create a new function `vote` to expose an entrypoint of the same name:

```rust
pub fn vote(&mut self, candidate: String) {

}
```

The only parameter (besides a mutable reference to `self`) is `candidate`, which denotes which candidate the caller would like to vote for.

Start off the function by checking if the current block time is greater than `end_block`. If it is, revert with the error `VotingEnded`:

```rust
if self.env().get_block_time() > self.end_block {
	self.env().revert(Error::VotingEnded);
}
```

Next, obtain the caller's `Address` as it will be used to determine if the calling account has already voted:

```rust
let caller: Address = self.env().caller();
```

Now prepare a `match` expression to get the dictionary value at the key `caller`:

```rust
match self.voters.get(caller) {
	Ok(Some(_)) => self.env().revert(Error::VoterAlreadyVoted),
	Ok(None) => {},
	Err(_) => self.env().revert(Error::ProbablyBecauseTheValueIsEmpty)
}
```

In this case, if a value exists, then it is known that the account has already voted, and execution can be reverted with `VoterAlreadyVoted`.
If no value exists, execution may continue.
If an error occurs, revert with `UnknownError`. ***Fix this***

Assuming execution isn't reverted due to the result of the `match` expression, the only thing left to do is record the user's vote and register them in `voters` as having voted.

Start by getting the selected candidate's current vote count:

```rust
let candidate_vote_count: u32 = self.candidate_votes.get_or_default(&candidate);
```

Then write this value, plus 1, to `candidate_votes`:

```rust
self.candidate_votes.set(&candidate, candidate_vote_count + 1u32);
```

Finally, add the calling account's `Address` to `voters` to restrict the user from voting again:

```rust
self.voters.set(caller, true)
```

## Testing

