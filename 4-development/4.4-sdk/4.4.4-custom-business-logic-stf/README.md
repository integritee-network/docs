# 4.4.4 Custom Business Logic / STF

When building your own solution, based on the [sidechain SDK](../4.4.1-sidechain-sdk.md) or the [offchain-worker SDK](../4.4.2-trusted-off-chain-worker.md), a key task is to integrate your own business logic (also called STF, state transition function). Here we will show you how that is done in the Integritee worker.

### **General Concept**

Introducing your own business logic is mostly done in the STF crate `ita-stf`. The `TrustedCall` and `TrustedGetter` structs are the declarations of the business logic and should be extended to match your use-case.



The template declaration contains some basic logic for transferring, shielding and unshielding funds.

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

Implementing the business logic is done in `stf_sgx.rs`, for each trusted call (`fn execute(..)`) and trusted getter (`fn get_state(..)`).

&#x20;

### **Using Substrate Pallets**

For developers who are used to writing substrate pallets, they can do so for the Integritee worker as well: Pallets can be imported and used inside the worker STF. This is possible thanks to our own adaptation of the substrate runtime to SGX (see [sgx-runtime repo](https://github.com/integritee-network/sgx-runtime)).

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

Code snippet taken from [here](https://github.com/integritee-network/worker/blob/a9a5afdb2de093de0062d7cb7ad302b8501e24a0/app-libs/stf/src/stf\_sgx.rs#L155).\


### **CLI Client**

The [CLI client](https://github.com/integritee-network/worker/tree/master/cli) in the worker repository is used as a driver for worker request (both direct and indirect invocation). So after you have implemented your business logic in the `stf` crate of the worker, you will want to extend the CLI client to support the new operations.

1.  Extend the command-line interface of the CLI client, using the [clap v3.0](https://docs.rs/clap/latest/clap/index.html) CLI parser, e.g. [here](https://github.com/integritee-network/worker/blob/a9a5afdb2de093de0062d7cb7ad302b8501e24a0/cli/src/trusted\_commands.rs#L92).

    ```rust
    #[derive(Subcommand)]
    pub enum TrustedCommands {
        /// generates a new incognito account for the given shard
        NewAccount,

        /// lists all incognito accounts in a given shard
        ListAccounts,

        /// send funds from one incognito account to another
        Transfer {
     	   /// sender's AccountId in ss58check format
     	   from: String,

     	   /// recipient's AccountId in ss58check format
     	   to: String,

     	   /// amount to be transferred
     	   amount: Balance,
        },
        ...
     }
     
    ```
2.  Translate the new command-line parameters into a corresponding `TrustedCall`, as is done for example for [`TrustedCall::balance_transfer`](https://github.com/integritee-network/worker/blob/a9a5afdb2de093de0062d7cb7ad302b8501e24a0/cli/src/trusted\_commands.rs#L225)`:`

    {% code overflow="wrap" %}
    ```rust
    fn transfer(cli: &Cli, trusted_args: &TrustedArgs, arg_from: &str, arg_to: &str, amount: &Balance) {
         let from = get_pair_from_str(trusted_args, arg_from);
         let to = get_accountid_from_str(arg_to);
         let (mrenclave, shard) = get_identifiers(trusted_args);
         let nonce = get_layer_two_nonce!(from, cli, trusted_args);
         
         let top: TrustedOperation = 
         TrustedCall::balance_transfer(from.public().into(), to, *amount)
     	    .sign(&KeyPair::Sr25519(from), nonce, &mrenclave, &shard)
     	    .into_trusted_operation(trusted_args.direct);

         let _ = perform_operation(cli, trusted_args, &top);
    }
    ```
    {% endcode %}

### **Example**

* ​[Encointer (advanced)](4.4.4.1-example-encointer.md)
