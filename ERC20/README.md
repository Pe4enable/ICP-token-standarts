# Analog of ERC20 on IC

# <h1> Not audited !!! </h1>

The ICRC-1 is a standard for Fungible Tokens on the Internet Computer.
ICRC-2 is an extension of the base ICRC-1 standard. ICRC-2 specifies a way for an account owner to delegate token transfers to a third party on the owner's behalf.

## Reference

https://github.com/dfinity/ICRC-1

## Description

## Data

### account

A `principal` can have multiple accounts. Each account of a `principal` is identified by a 32-byte string called `subaccount`. Therefore an account corresponds to a pair `(principal, subaccount)`.

The account identified by the subaccount with all bytes set to 0 is the _default account_ of the `principal`.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner : principal; subaccount : opt Subaccount; };
```

## Methods

### icrc1_name <span id="name_method"></span>

Returns the name of the token (e.g., `MyToken`).

```candid "Methods" +=
icrc1_name : () -> (text) query;
```

### icrc1_symbol <span id="symbol_method"></span>

Returns the symbol of the token (e.g., `ICP`).

```candid "Methods" +=
icrc1_symbol : () -> (text) query;
```

### icrc1_decimals <span id="decimals_method"></span>

Returns the number of decimals the token uses (e.g., `8` means to divide the token amount by `100000000` to get its user representation).

```candid "Methods" +=
icrc1_decimals : () -> (nat8) query;
```

### icrc1_fee <span id="fee_method"></span>

Returns the default transfer fee.

```candid "Methods" +=
icrc1_fee : () -> (nat) query;
```

### icrc1_metadata <span id="metadata_method"></span>

Returns the list of metadata entries for this ledger.
See the "Metadata" section below.

```candid "Type definitions" +=
type Value = variant { Nat : nat; Int : int; Text : text; Blob : blob };
```

```candid "Methods" +=
icrc1_metadata : () -> (vec record { text; Value }) query;
```

### icrc1_total_supply

Returns the total number of tokens on all accounts except for the [minting account](#minting_account).

```candid "Methods" +=
icrc1_total_supply : () -> (nat) query;
```

### icrc1_minting_account

Returns the [minting account](#minting_account) if this ledger supports minting and burning tokens.

```candid "Methods" +=
icrc1_minting_account : () -> (opt Account) query;
```

### icrc1_balance_of

Returns the balance of the account given as an argument.

```candid "Methods" +=
icrc1_balance_of : (Account) -> (nat) query;
```

### icrc1_transfer <span id="transfer_method"></span>

Transfers `amount` of tokens from account `record { of = caller; subaccount = from_subaccount }` to the `to` account.
The caller pays `fee` tokens for the transfer.

```candid "Type definitions" +=
type TransferArgs = record {
    from_subaccount : opt Subaccount;
    to : Account;
    amount : nat;
    fee : opt nat;
    memo : opt blob;
    created_at_time : opt nat64;
};

type TransferError = variant {
    BadFee : record { expected_fee : nat };
    BadBurn : record { min_burn_amount : nat };
    InsufficientFunds : record { balance : nat };
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc1_transfer : (TransferArgs) -> (variant { Ok: nat; Err: TransferError; });
```

The caller pays the `fee`.
If the caller does not set the `fee` argument, the ledger applies the default transfer fee.
If the `fee` argument does not agree with the ledger fee, the ledger MUST return `variant { BadFee = record { expected_fee = ... } }` error.

The `memo` parameter is an arbitrary blob that has no meaning to the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction_deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

The result is either the transaction index of the transfer or an error.

### icrc1_supported_standards

Returns the list of standards this ledger implements.
See the ["Extensions"](#extensions) section below.

```candid "Methods" +=
icrc1_supported_standards : () -> (vec record { name : text; url : text }) query;
```

The result of the call should always have at least one entry,

```candid
record { name = "ICRC-1"; url = "https://github.com/dfinity/ICRC-1" }
```

## Extensions <span id="extensions"></span>

The base standard intentionally excludes some ledger functions essential for building a rich DeFi ecosystem, for example:

  - Reliable transaction notifications for smart contracts.
  - The block structure and the interface for fetching blocks.
  - Pre-signed transactions.

The standard defines the `icrc1_supported_standards` endpoint to accommodate these and other future extensions.
This endpoint returns names of all specifications (e.g., `"ICRC-42"` or `"DIP-20"`) implemented by the ledger.

## Metadata

A ledger can expose metadata to simplify integration with wallets and improve user experience.
The client can use the [`icrc1_metadata`](#metadata_method) method to fetch the metadata entries. 
All the metadata entries are optional.

### Key format

The metadata keys are arbitrary Unicode strings and must follow the pattern `<namespace>:<key>`, where `<namespace>` is a string not containing colons.
Namespace `icrc1` is reserved for keys defined in this standard.

### Standard metadata entries
| Key | Semantics | Example value
| --- | ------------- | --------- |
| `icrc1:symbol` | The token currency code (see [ISO-4217](https://en.wikipedia.org/wiki/ISO_4217)). When present, should be the same as the result of the [`icrc1_symbol`](#symbol_method) query call. | `variant { Text = "XTKN" }` | 
| `icrc1:name` | The name of the token. When present, should be the same as the result of the [`icrc1_name`](#name_method) query call. | `variant { Text = "Test Token" }` | 
| `icrc1:decimals` |  The number of decimals the token uses. For example, 8 means to divide the token amount by 10<sup>8</sup> to get its user representation. When present, should be the same as the result of the [`icrc1_decimals`](#decimals_method) query call. | `variant { Nat = 8 }` |
| `icrc1:fee` | The default transfer fee. When present, should be the same as the result of the [`icrc1_fee`](#fee_method) query call. |  `variant { Nat = 10_000 }` |
| `icrc1:logo` | The URL of the token logo. The value can contain the actual image if it's a [Data URL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs).  | `variant { Text = "data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iMSIgaGVpZ2h0PSIxIiB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciPjxyZWN0IHdpZHRoPSIxMDAlIiBoZWlnaHQ9IjEwMCUiIGZpbGw9InJlZCIvPjwvc3ZnPg==" }` | 


## Transaction deduplication <span id="transaction_deduplication"></span>

Consider the following scenario:

  1. An agent sends a transaction to an ICRC-1 ledger hosted on the IC.
  2. The ledger accepts the transaction.
  3. The agent loses the network connection for several minutes and cannot learn about the outcome of the transaction.

An ICRC-1 ledger SHOULD implement transfer deduplication to simplify the error recovery for agents.
The deduplication covers all transactions submitted within a pre-configured time window `TX_WINDOW` (for example, last 24 hours).
The ledger MAY extend the deduplication window into the future by the `PERMITTED_DRIFT` parameter (for example, 2 minutes) to account for the time drift between the client and the Internet Computer.

The client can control the deduplication algorithm using the `created_at_time` and `memo` fields of the [`transfer`](#transfer_method) call argument:
  * The `created_at_time` field sets the transaction construction time as the number of nanoseconds from the UNIX epoch in the UTC timezone.
  * The `memo` field does not have any meaning to the ledger, except that the ledger will not deduplicate transfers with different values of the `memo` field.

The ledger SHOULD use the following algorithm for transaction deduplication if the client set the `created_at_time` field:
  * If `created_at_time` is set and is _before_ `time() - TX_WINDOW - PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { TooOld }` error.
  * If `created_at_time` is set and is _after_ `time() + PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { CreatedInFuture = record { ledger_time = ... } }` error.
  * If the ledger observed a structurally equal transfer payload (i.e., all the transfer argument fields and the caller have the same values) at transaction with index `i`, it should return `variant { Duplicate = record { duplicate_of = i } }`.
  * Otherwise, the transfer is a new transaction.

If the client did not set the `created_at_time` field, the ledger SHOULD NOT deduplicate the transaction.

## Minting account <span id="minting_account"></span>

The minting account is a unique account that can create new tokens and acts as the receiver of burnt tokens.

Transfers _from_ the minting account act as _mint_ transactions depositing fresh tokens on the destination account.
Mint transactions have no fee.

Transfers _to_ the minting account act as _burn_ transactions, removing tokens from the token supply.
Burn transactions have no fee but might have minimal burn amount requirements.
If the client tries to burn an amount that is too small, the ledger SHOULD reply with

```
variant { Err = variant { BadBurn = record { min_burn_amount = ... } } }
```

The minting account is also the receiver of the fees burnt in regular transfers.

<!--
```candid ICRC-1.did +=
<<<Type definitions>>>

service : {
  <<<Methods>>>
}
```
-->

The approve and transfer-from flow is a 2-step process.
1. Account owner Alice entitles Bob to transfer up to X tokens from her account A by calling the `icrc2_approve` method on the ledger.
2. Bob can transfer up to X tokens from account A to any account by calling the `icrc2_transfer_from` method on the ledger as if A was Bob's account B.
   The number of transfers Bob can initiate from account A is not limited as long as the total amount spent is below X.

Approvals are not transitive: if Alice approves transfers from her account A to Bob's account B, and Bob approves transfers from his account B to Eva's account E, Eva cannot withdraw tokens from Alice's account through Bob's approval.

## Motivation

The approve-transfer-from pattern became popular in the Ethereum ecosystem thanks to the [ERC-20](https://ethereum.org/en/developers/docs/standards/tokens/erc-20/) token standard.
This interface enables new application capabilities:

  1. Recurring payments.
     Alice can approve a large amount to Bob in advance, allowing Bob to make periodic transfers in smaller installments.
     Real-world examples include subscription services and rents.

  2. Uncertain transfer amounts.
     In some applications, such as automatic trading services, the exact price of goods is unknown in advance.
     With approve-transfer-from flow, Alice can allow Bob to trade securities on Alice's behalf, buying/selling at yet-unknown price up to a specified limit.

## Specification

> The keywords "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119.

**Canisters implementing the `ICRC-2` standard MUST implement all the functions in the `ICRC-1` interface**

**Canisters implementing the `ICRC-2` standard MUST include `ICRC-2` in the list returned by the `icrc1_supported_standards` method**

## Methods

```candid "Type definitions" +=
type Account = record {
    owner : principal;
    subaccount : opt blob;
};
```

### icrc2_approve

This method entitles the `spender` to transfer token `amount` on behalf of the caller from account `{ owner = caller; subaccount = from_subaccount }`.
The number of transfers the `spender` can initiate from the caller's account is unlimited as long as the total amounts and fees of these transfers do not exceed the allowance.
The caller does not need to have the full token `amount` on the specified account for the approval to succeed, just enough tokens to pay the approval fee.
The call resets the allowance and the expiration date for the `spender` account to the given values.

The ledger SHOULD reject the call if the spender account owner is equal to the source account owner.

If the `expected_allowance` field is set, the ledger MUST ensure that the current allowance for the `spender` from the caller's account is equal to the given value and return the `AllowanceChanged` error otherwise.

The ledger MAY cap the allowance if it is too large (for example, larger than the total token supply).
For example, if there are only 100 tokens, and the ledger receives an approval for 120 tokens, the ledger may cap the allowance to 100.

```candid "Methods" +=
icrc2_approve : (ApproveArgs) -> (variant { Ok : nat; Err : ApproveError });
```

```candid "Type definitions" +=
type ApproveArgs = record {
    from_subaccount : opt blob;
    spender : Account;
    amount : nat;
    expected_allowance : opt nat;
    expires_at : opt nat64;
    fee : opt nat;
    memo : opt blob;
    created_at_time : opt nat64;
};

type ApproveError = variant {
    BadFee : record { expected_fee : nat };
    // The caller does not have enough funds to pay the approval fee.
    InsufficientFunds : record { balance : nat };
    // The caller specified the [expected_allowance] field, and the current
    // allowance did not match the given value.
    AllowanceChanged : record { current_allowance : nat };
    // The approval request expired before the ledger had a chance to apply it.
    Expired : record { ledger_time : nat64; };
    TooOld;
    CreatedInFuture: record { ledger_time : nat64 };
    Duplicate : record { duplicate_of : nat };
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text };
};
```

#### Preconditions

* The caller has enough fees on the `{ owner = caller; subaccount = from_subaccount }` account to pay the approval fee.
* If the `expires_at` field is set, it's greater than the current ledger time.
* If the `expected_allowance` field is set, it's equal to the current allowance for the `spender`.

#### Postconditions

* The `spender`'s allowance for the `{ owner = caller; subaccount = from_subaccount }` is equal to the given `amount`.

### icrc2_transfer_from

Transfers a token amount from the `from` account to the `to` account using the allowance of the spender's account (`SpenderAccount = { owner = caller; subaccount = spender_subaccount }`).
The ledger draws the fees from the `from` account.

```candid "Methods" +=
icrc2_transfer_from : (TransferFromArgs) -> (variant { Ok : nat; Err : TransferFromError });
```

```candid "Type definitions" +=
type TransferFromError = variant {
    BadFee : record { expected_fee : nat };
    BadBurn : record { min_burn_amount : nat };
    // The [from] account does not hold enough funds for the transfer.
    InsufficientFunds : record { balance : nat };
    // The caller exceeded its allowance.
    InsufficientAllowance : record { allowance : nat };
    TooOld;
    CreatedInFuture: record { ledger_time : nat64 };
    Duplicate : record { duplicate_of : nat };
    TemporarilyUnavailable;
    GenericError : record { error_code : nat; message : text };
};

type TransferFromArgs = record {
    spender_subaccount : opt blob;
    from : Account;
    to : Account;
    amount : nat;
    fee : opt nat;
    memo : opt blob;
    created_at_time : opt nat64;
};
```

#### Preconditions
 
 * The allowance for the `SpenderAccount` from the `from` account is large enough to cover the transfer amount and the fees
   (`icrc2_allowance({ account = from; spender = SpenderAccount }).allowance >= amount + fee`). 
   Otherwise, the ledger MUST return an `InsufficientAllowance` error.

* The `from` account holds enough funds to cover the transfer amount and the fees.
  (`icrc1_balance_of(from) >= amount + fee`).
  Otherwise, the ledger MUST return an `InsufficientFunds` error.

 #### Postconditions

 * If the `from` account is not equal to the `SpenderAccount`, the `(from, SpenderAccount)` allowance decreases by the transfer amount and the fees.
 * The ledger debited the specified `amount` of tokens and fees from the `from` account.
 * The ledger credited the specified `amount` to the `to` account.

### icrc2_allowance

Returns the token allowance that the `spender` account can transfer from the specified `account`, and the expiration time for that allowance, if any.
If there is no active approval, the ledger MUST return `{ allowance = 0; expires_at = null }`.

```candid "Methods" +=
icrc2_allowance : (AllowanceArgs) -> (Allowance) query;
```
```candid "Type definitions" +=
type AllowanceArgs = record {
    account : Account;
    spender : Account;
};

type Allowance = record {
  allowance : nat;
  expires_at : opt nat64;
}
```

### icrc1_supported_standards

Returns the list of standards this ledger supports.
Any ledger supporting `ICRC-2` MUST include a record with the `name` field equal to `"ICRC-2"` in that list.

```candid "Methods" +=
icrc1_supported_standards : () -> (vec record { name : text; url : text }) query;
```

## Examples

### Alice deposits tokens to canister C

1. Alice wants to deposit 100 tokens on an `ICRC-2` ledger to canister C.
2. Alice calls `icrc2_approve` with `spender` set to the canister's default account (`{ owner = C; subaccount = null}`) and `amount` set to the token amount she wants to deposit (100) plus the transfer fee.
3. Alice can then call some `deposit` method on the canister, which calls `icrc2_transfer_from` with `from` set to Alice's (the caller) account, `to` set to the canister's account, and `amount` set to the token amount she wants to deposit (100).
4. The canister can now determine from the result of the call whether the transfer was successful.
   If it was successful, the canister can now safely commit the deposit to state and know that the tokens are in its account.

### Canister C transfers tokens from Alice's account to Bob's account, on Alice's behalf

1. Canister C wants to transfer 100 tokens on an `ICRC-2` ledger from Alice's account to Bob's account.
2. Alice previously approved canister C to transfer tokens on her behalf by calling `icrc2_approve` with `spender` set to the canister's default account (`{ owner = C; subaccount = null }`) and `amount` set to the token amount she wants to allow (100) plus the transfer fee.
3. During some update call, the canister can now call `icrc2_transfer_from` with `from` set to Alice's account, `to` set to Bob's account, and `amount` set to the token amount she wants to transfer (100).
4. Once the call completes successfully, Bob has 100 extra tokens on his account, and Alice has 100 (plus the fee) tokens less in her account.

### Alice removes her allowance for canister C

1. Alice wants to remove her allowance of 100 tokens on an `ICRC-2` ledger for canister C.
2. Alice calls `icrc2_approve` on the ledger with `spender` set to the canister's default account (`{ owner = C; subaccount = null }`) and `amount` set to 0.
3. The canister can no longer transfer tokens on Alice's behalf.

### Alice atomically removes her allowance for canister C

1. Alice wants to remove her allowance of 100 tokens on an `ICRC-2` ledger for canister C.
2. Alice calls `icrc2_approve` on the ledger with `spender` set to the canister's default account (`{ owner = C; subaccount = null }`), `amount` set to 0, and `expected_allowance` set to 100 tokens.
3. If the call succeeds, the allowance got removed successfully.
   An `AllowanceChanged` error would indicate that canister C used some of the allowance before Alice's call completed.

<!--
```candid ICRC-2.did +=
<<<Type definitions>>>

service : {
  <<<Methods>>>
}
```
-->