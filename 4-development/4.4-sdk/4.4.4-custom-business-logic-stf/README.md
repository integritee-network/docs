# 4.4.4 Custom Business Logic / STF

When building your own solution, based on the [sidechain SDK](../4.4.1-sidechain-sdk.md) or the [offchain-worker SDK](../4.4.2-trusted-off-chain-worker.md), a key task is to integrate your own business logic (also called STF, state transition function). Here we will show you how that is done in the Integritee worker.

### **General Concept**

Introducing your own business logic is mostly done in the STF crate `ita-stf`. The template declaration contains some basic logic for transferring, shielding and unshielding funds, which you can expand according to your needs.

```rust
pub enum TrustedCall {
	balance_set_balance(...),
	balance_transfer(...),
	balance_unshield(...),
	balance_shield(...)
}
```

```rust
pub enum TrustedGetter {
	free_balance(...),
	reserved_balance(...),
	nonce(...),
}
```

The `TrustedCall` and `TrustedGetter` structs are the declarations of the business logic interfaces and should be extended to match your use-case by:

1. Adding an enum variant of your call/getter
2. Implementing what should be done for your enum variant in the [ExecuteCall](https://github.com/integritee-network/worker/blob/b54c825fb13fe99f77fe549798a6c63853d81c34/app-libs/stf/src/trusted\_call.rs#L171) or [ExecuteGetter](https://github.com/integritee-network/worker/blob/b54c825fb13fe99f77fe549798a6c63853d81c34/app-libs/stf/src/getter.rs#L113) block.

How to do that will be shown below.

### **Using Substrate Pallets**

For developers who are used to writing substrate pallets, they can do so for the Integritee worker as well: Pallets can be imported and used inside the worker STF. This is possible thanks to our own adaptation of the substrate runtime to SGX (see the [sgx-runtime](https://github.com/integritee-network/worker/tree/master/app-libs/sgx-runtime) crate).

An example is the existing implementation of `balance_transfer`, using the standard substrate `Balance` pallet (imported [here](https://github.com/integritee-network/sgx-runtime/blob/cefb6991a5ddc7f8a1139da8f48a49b6379113df/runtime/src/lib.rs#L53)):

{% code overflow="wrap" %}
```rust
TrustedCall::balance_transfer(from, to, value) => {
    let origin = sgx_runtime::Origin::signed(from.clone());

    // ...
        
    sgx_runtime::BalancesCall::<Runtime>::transfer {
        dest: MultiAddress::Id(to),
		value,
	}
	.dispatch_bypass_filter(origin)
	.map_err(|e| {
		StfError::Dispatch(format!("Balance Transfer error: {:?}", e.error))
	})?;

	Ok(())
},
```
{% endcode %}

This code snippet is from the `app-libs/stf` directory, and more specifically [here](https://github.com/integritee-network/worker/blob/b54c825fb13fe99f77fe549798a6c63853d81c34/app-libs/stf/src/trusted\_call.rs#L202).

### Using your own code

You don't need to use a substrate pallet to implement some business logic there, you could also implement your own function like:

```rust
// This should be put in the `ExecuteCall` implementation block.
TrustedCall::set_magic_value(value) => {
    // All the code here is executed in the scope of the `state.with` function, see:
    //
    //     https://github.com/integritee-network/worker/blob/master/app-libs/stf/src/stf_sgx.rs#L130
    // 
    // This gives you access to the underlying database via the `sp_io::storage` module.
    
    // Write value to state, which will be encrypted afterwards
    sp_io::storage::set(
        &storage_value_key("MyModule", "MyMagicValue"),
        &value.encode(),
    );
    
    Ok(())
},


// This should be put in the `ExecuteGetter` implementation block.
TrustedGetter::magic_value => {
    // All the code here is executed in the scope of the `state.with` function, see:
    //
    //     https://github.com/integritee-network/worker/blob/master/app-libs/stf/src/stf_sgx.rs#L146
    // 
    // This gives you access to the underlying database via the `sp_io::storage` module.
    
    // Read value from state, which has been dencrypted before the trusted getter execution.
    let maybe_value_encoded =
	sp_io::storage::get(&storage_value_key("MyModule", "MyMagicValue"));

    match maybe_value_encoded {
        Some(ref value) => debug!("Found Magic Value: {:?}", value),
	None => debug!("Magic Value not found!"),
    };
    
    maybe_value_encoded
},
```

### **CLI Client**

The [CLI client](https://github.com/integritee-network/worker/tree/master/cli) in the worker repository is used as a driver for worker request (both direct and indirect invocation). So after you have implemented your business logic in the `stf` crate of the worker, you will want to extend the CLI client to support the new operations.&#x20;

#### Main CLI

The overarching CLI is defined in the [commands.rs](https://github.com/integritee-network/worker/blob/master/cli/src/commands.rs), and looks as follows:

We observe that we have divided the commands to different categories, where the `BaseCommand` contains regular commands talking to the service or the parentchain node, the `TrustedCli` contains the subcommands talking to the enclave, and the `OracleSubCommand` is only for oracle related stuff.

```rust
use crate::{base_cli::BaseCommand, trusted_cli::TrustedCli, Cli};
use clap::Subcommand;

#[cfg(feature = "teeracle")]
use crate::oracle::OracleCommand;

#[derive(Subcommand)]
pub enum Commands {
	#[clap(flatten)]
	Base(BaseCommand),

	/// trusted calls to worker enclave
	#[clap(after_help = "stf subcommands depend on the stf crate this has been built against")]
	Trusted(TrustedCli),

	/// Subcommands for the oracle.
	#[cfg(feature = "teeracle")]
	#[clap(subcommand)]
	Oracle(OracleCommand),
}

pub fn match_command(cli: &Cli) {
	match &cli.command {
		Commands::Base(cmd) => cmd.run(cli),
		Commands::Trusted(trusted_cli) => trusted_cli.run(cli),
		#[cfg(feature = "teeracle")]
		Commands::Oracle(cmd) => cmd.run(cli),
	};
}
```

#### Extending the TrustedBaseCli

The overarching CLI proxies the arguments to the [TrustedBaseCli](https://github.com/integritee-network/worker/blob/master/cli/src/trusted\_base\_cli/mod.rs), which looks like this:

```rust
#[derive(Subcommand)]
pub enum TrustedBaseCli {
	/// generates a new incognito account for the given shard
	NewAccount,

	/// lists all incognito accounts in a given shard
	ListAccounts,

	/// send funds from one incognito account to another
	Transfer(TransferCommand),

	/// ROOT call to set some account balance to an arbitrary number
	SetBalance(SetBalanceCommand),

	/// query balance for incognito account in keystore
	Balance(BalanceCommand),

	/// Transfer funds from an incognito account to an parentchain account
	UnshieldFunds(UnshieldFundsCommand),
}

impl TrustedBaseCli {
	pub fn run(&self, cli: &Cli, trusted_cli: &TrustedCli) {
		match self {
			TrustedBaseCli::NewAccount => new_account(trusted_cli),
			TrustedBaseCli::ListAccounts => list_accounts(trusted_cli),
			TrustedBaseCli::Transfer(cmd) => cmd.run(cli, trusted_cli),
			TrustedBaseCli::SetBalance(cmd) => cmd.run(cli, trusted_cli),
			TrustedBaseCli::Balance(cmd) => cmd.run(cli, trusted_cli),
			TrustedBaseCli::UnshieldFunds(cmd) => cmd.run(cli, trusted_cli),
		}
	}
}
```

As an example, we will look at the [TransferCommand](https://github.com/integritee-network/worker/blob/master/cli/src/trusted\_base\_cli/commands/transfer.rs).

```rust
#[derive(Parser)]
pub struct TransferCommand {
	/// sender's AccountId in ss58check format
	from: String,

	/// recipient's AccountId in ss58check format
	to: String,

	/// amount to be transferred
	amount: Balance,
}

impl TransferCommand {
	pub(crate) fn run(&self, cli: &Cli, trusted_cli: &TrustedCli) {
		let from = get_pair_from_str(trusted_cli, &self.from);
		let to = get_accountid_from_str(&self.to);
		info!("from ss58 is {}", from.public().to_ss58check());
		info!("to ss58 is {}", to.to_ss58check());

		println!("send trusted call transfer from {} to {}: {}", from.public(), to, self.amount);
		let (mrenclave, shard) = get_identifiers(trusted_cli);
		let nonce = get_layer_two_nonce!(from, cli, trusted_cli);
		
		/// Construct the `TrustedOperation` containing the `TrustedCall` that we have
		// defined as interface to the business logic above.
		let top: TrustedOperation =
			TrustedCall::balance_transfer(from.public().into(), to, self.amount)
				.sign(&KeyPair::Sr25519(Box::new(from)), nonce, &mrenclave, &shard)
				.into_trusted_operation(trusted_cli.direct);
				
		
		// Send an RPC request to the enclave and execute the trusted call/getter
		// inside the STF and return the result.
		let _ = perform_trusted_operation(cli, trusted_cli, &top);
		info!("trusted call transfer executed");
	}
```

Conclusively, adding a new trusted command looks like this:

1. Write the `MyCustomCommand` struct and define the arguments it should take.
2. Implement the `run` function for it.
3. Extend the `TrustedBaseCli` with another enum variant `CustomCommand(MyCustomCommand)`

That's it. Simple, isn't it?

### **Example**

* ​[Encointer (advanced)](4.4.4.1-example-encointer.md)
