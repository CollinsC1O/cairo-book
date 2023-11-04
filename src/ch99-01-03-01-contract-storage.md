# Contract Storage

The most common way for interacting with a contract’s storage is through storage variables. As stated previously, storage variables allow you to store data that will be stored in the contract's storage that is itself stored on the blockchain. These data are persistent and can be accessed and modified anytime once the contract is deployed.

Storage variables in Starknet contracts are stored in a special struct called `Storage`:

```rust, noplayground
{{#rustdoc_include ../listings/ch99-starknet-smart-contracts/listing_99_03_example_contract/src/lib.cairo:storage}}
```

<span class="caption">A Storage Struct</span>

The storage struct is a [struct](./ch05-00-using-structs-to-structure-related-data.md) like any other,
except that it **must** be annotated with `#[storage]`. This annotation tells the compiler to generate the required code to interact with the blockchain state, and allows you to read and write data from and to storage. Moreover, this allows you to define storage mappings using the `LegacyMap` type.

Each variable stored in the storage struct is stored in a different location in the contract's storage. The storage address of a variable is determined by the variable's name, and the eventual keys of the variable if it is a [mapping](#storing-mappings).

The address of a storage variable is computed as follows:

- If the variable is a single value (not a mapping), the address is the `sn_keccak` hash of the ASCII encoding of the variable's name. `sn_keccak` is Starknet's version of the Keccak256 hash function, whose output is truncated to 250 bits.
- If the variable is a [mapping](#storing-mappings), the address of the value at key `k_1,...,k_n` is `h(...h(h(sn_keccak(variable_name),k_1),k_2),...,k_n)` where ℎ is the Pedersen hash and the final value is taken `mod (2^251) − 256`.
- If it is a mapping to complex values (e.g., tuples or structs), then this complex value lies in a continuous segment starting from the address calculated in the previous point. Note that 256 field elements is the current limitation on the maximal size of a complex storage value.

## Interacting with Storage Variables

Variables stored in the storage struct can be accessed and modified using the `read` and `write` functions, and you can get their address in storage using the `addr` function. These functions are automatically generated by the compiler for each storage variable.

To read the value of the `owner` storage variable, which is a single value, we call the `read` function on the `owner` variable, passing in no parameters.

```rust, noplayground
{{#include ../listings/ch99-starknet-smart-contracts/listing_99_03_example_contract/src/lib.cairo:read_owner}}
```

<span class="caption">Calling the `read` function on the `owner` variable</span>

To read the value of the storage variable `names`, which is a mapping from `ContractAddress` to `felt252`, we call the `read` function on the `names` variable, passing in the key `address` as a parameter. If the mapping had more than one key, we would pass in the other keys as parameters as well.

```rust, noplayground
{{#include ../listings/ch99-starknet-smart-contracts/listing_99_03_example_contract/src/lib.cairo:read}}
```

<span class="caption">Calling the `read` function on the `names` variable</span>

To write a value to a storage variable, we call the `write` function passing in the eventual keys the value as arguments. As with the `read` function, the number of arguments depends on the number of keys - here, we only pass in the value to write to the `owner` variable as it is a simple variable.

```rust, noplayground
{{#include ../listings/ch99-starknet-smart-contracts/listing_99_03_example_contract/src/lib.cairo:write_owner}}
```

<span class="caption">Calling the `write` function on the `owner` variable</span>

```rust, noplayground
{{#include ../listings/ch99-starknet-smart-contracts/listing_99_03_example_contract/src/lib.cairo:write}}
```

<span class="caption">Calling the `write` function on the `names` variable</span>

## Storing custom types

The `Store` trait, defined in the `starknet::storage_access` module, is used to specify how a type should be stored in storage. In order for a type to be stored in storage, it must implement the `Store` trait. Most types from the core library, such as unsigned integers (`u8`, `u128`, `u256`...), `felt252`, `bool`, `ContractAddress`, etc. implement the `Store` trait and can thus be stored without further action.

But what if you wanted to store a type that you defined yourself, such as an enum or a struct? In that case, you have to explicitly tell the compiler how to store this type.

In our example, we want to store a `Person` struct in storage, which is possible by implementing the `Store` trait for the `Person` type. This can be achieved by simply adding a `#[derive(starknet::Store)]` attribute on top of our struct definition.

```rust, noplayground
{{#include ../listings/ch99-starknet-smart-contracts/listing_99_03_example_contract/src/lib.cairo:person}}
```

### Structs storage layout

On Starknet, structs are stored in storage as a sequence of primitive types.
The elements of the struct are stored in the same order as they are defined in the struct definition. The first element of the struct is stored at the base address of the struct, which is computed as specified previously and can be obtained by calling `var.address()`, and subsequent elements are stored at addresses contiguous to the first element.
For example, the storage layout for the `owner` variable of type `Person` will result in the following layout:

| Fields  | Address            |
| ------- | ------------------ |
| name    | owner.address()    |
| address | owner.address() +1 |

## Storing mappings

Mappings are a key-value data structure that you can use to store data within a smart contract. They are essentially hash tables that allow you to associate a unique key with a corresponding value. Mappings are also useful to store sets of data, as it's impossible to store arrays in storage.

A mapping is a variable of type `LegacyMap`, in which the key and value types are specified within angular brackets `<>`.
It is important to note that the `LegacyMap` type can **only** be used inside the `Storage` struct, and cannot be used as a local variable.

You can also create more complex mappings than that; you can find one in Listing 99-2bis like the popular `allowances` storage variable in the ERC20 Standard which maps the `owner` and `spender` to the `allowance` using tuples:

```rust,noplayground
{{#include ../listings/ch99-starknet-smart-contracts/no_listing_01_storage_mapping/src/lib.cairo:here}}
```

<span class="caption">Listing 99-2bis: Storing mappings</span>

In mappings, the address of the value at key `k_1,...,k_n` is `h(...h(h(sn_keccak(variable_name),k_1),k_2),...,k_n)` where ℎ is the Pedersen hash and the final value is taken `mod (2^251) − 256`.

If the key of the mapping is a struct, each element of the struct constitutes a key. Moreover, the struct should implement the Hash trait, which can be derived with the `#[derive(Hash)]` attribute. For example, if you have struct with two fields, the address will be `h(h(sn_keccak(variable_name),k_1),k_2)` - where `k_1` and `k_2` are the values of the two fields of the struct.

Similarly, in the case of a nested mapping such as `LegacyMap((ContractAddress, ContractAddress), u8)`, the address will be computed in the same way: `h(h(sn_keccak(variable_name),k_1),k_2)`.

You can learn more about the contract storage layout in the [Starknet Documentation](https://docs.starknet.io/documentation/architecture_and_concepts/Contracts/contract-storage/#storage_variables)