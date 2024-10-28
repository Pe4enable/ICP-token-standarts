# Analog of ERC721 on IC

# <h1> Not audited !!! </h1>

ICRC-7 is the minimal standard for the implementation of Non-Fungible Tokens (NFTs) on the Internet Computer.
ICRC-37 is an extension of the base ICRC-7 standard. 

## Reference

https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md


## Data Representation

This section specifies the core principles of data representation used in this standard.

### Accounts

A `principal` can have multiple accounts. Each account of a `principal` is identified by a 32-byte string called `subaccount`. Therefore, an account corresponds to a pair `(principal, subaccount)`.

The account identified by the subaccount with all bytes set to 0 is the *default account* of the `principal`.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner : principal; subaccount : opt Subaccount };
```

The canonical textual representation of the account follows the [definition in ICRC-1](https://github.com/dfinity/ICRC-1/blob/main/standards/ICRC-1/TextualEncoding.md). ICRC-7 accounts have the same structure and follow the same overall principles as ICRC-1 accounts.

ICRC-7 views the ICRC-1 `Account` as a primary concept, meaning that operations like transfers (or approvals as defined in ICRC-37) always refer to the full account and not only the principal part thereof. Thus, some methods comprise an extra optional `from_subaccount` or `spender_subaccount` parameter that together with the caller form an account to perform the respective operation on. Leaving such subaccount parameter `null` always has the semantics of referring to the default subaccount comprised of all zeroes.

### Token Identifiers

Tokens in ICRC-7 are identified through _token identifiers_, or _token ids_. A token id is a natural number value and thus unbounded in length. Token identifiers do not need to be allocated in a contiguous manner. Non-contiguous, i.e., sparse, allocations are, for example, useful for mapping string-based identifiers to token ids, which is, for example, important for making other NFT standards that use strings as token identifiers compatible with ICRC-7.

## Methods

### Generally-Applicable Specification

We next outline general aspects of the specification and behaviour of query and update calls defined in this standard. Those general aspects are not repeated with the specification of every method, but specified once for all query and update calls in this section.

#### Batch Update Methods

The methods that have at most one result per input value are modeled as *batch methods*, i.e., they operate on a vector of inputs and return a vector of outputs. The elements of the output are sorted in the same order as the elements of the input, meaning that the `i`-the element in the result is the reponse to the `i`-th element in the request. We call this property of the arguments "positional". The response may have fewer elements than the request in case processing has stopped through a batch processing error that prevents it from moving forward. In case of such batch processing error, the element which caused the batch processing to terminate receives an error response with the batch processing error. This element can not have a specific per-element error as it expresses the batch error. This element need not be the element with the highest index in the response as processing of requests can be concurrent in an implementation and any element earlier in the request may cause the batch processing failure.

The response of a batch method may be shorter than the request and then contain only responses to a prefix of the request vector. This happens when the processing is terminated due to an error. However, due to the ordering requirements of the response elements w.r.t. the request elements (positional arguments), the response must be contiguous, possibly containing `null` elements, i.e., it contains response elements to a contiguous prefix of the request vector.

The standard does not impose any constraints on aspects such as no duplicate token ids being contained in the elements of a request batch. Rather, each element is independent of the others and its execution may succeed or lead to an error. We do not impose any constraints on the sequence of processing of the elements of a batch in order to not have undue constraints on the implementation in terms of performance. A client SHOULD not assume any specific sequence of processing of batch elements. I.e., if a client intends to make dependent transactions, e.g., to move a token from its current subaccount to a specific "vendor subaccount" and from there transfer it to a customer, those two operations should be part of subsequent batches to assure both transfers to complete without assumptions on the implementation of the ledger.

Note that the items in a batch are processed independently of each other and processing can independently succeed or fail. This choice does not impose relevant constraints on the ledger implementation. The only constraint resulting from this is that the response must contain response items up to the largest element index processing of which has been initiated by the ledger, regardless of its result. The response items following this highest-index processed request item can be left out.

The API style we employ for batch APIs is simple, does not repeat request information in the response, and does not unnecessarily constrain the implementation, i.e., permits highly-concurrent implementations. On the client side it has no major drawbacks as it is straightforward to associated the corresponding request data with the responses by using positional alignment of the request and response vectors.

The guiding principle is to have all suitable update methods be batch methods, i.e., all update calls that have at most one response per request.

For batch update calls, each element in the request for which processing has been attempted, i.e., started, regardless of the success thereof, needs to contain a non-null response at the corresponding index. Elements for which processing has not been attempted, may contain a `null` response. The response vector may contain responses only to a prefix of the request vector, with further non-processed elements being omitted from the response.

Update calls, i.e., methods that modify the state of the ledger, always have responses that comprise transaction indices in the success case. Such a transaction index is an index into the chain of blocks containing the transaction history of this ledger. The details of how to access the transaction history of ICRC-7 ledgers is not part of the ICRC-7 standard, but will use the separately published [ICRC-3](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3) standard that specifies access to the block log of ICRC ledgers.

*Example*

Consider an input comprising the batch of transactions `A, B, C, D, E, F, G, H`, each of the items being one operation to perform and letters abstract the request items in the example.
```
Input : [A, B, C, D, E, F, G, H];
```
Depending on the concurrent execution of the individual operations, there may be different outcomes:

1. Assume an error that prevents further processing has occurred while processing `D`, with successfully processed `A`. Processing of `B` and `C` has not been initated yet, therefore they receive `null` responses.
  * Output: `[opt #Ok(5), null ,null, opt #Err(#GenericBatchError(...)];`
2. Assume an error that prevents further processing has occurred while processing `B` with successfully processed `A` and `D`, but processing for `C` has not been initiated. `A` and `D` receive a success response with their transaction id, `B` the batch error, and `C` is filled with a `null`:
  * Output: `[opt #Ok(5), opt #Err(#GenericBatchError(...), null , opt #Ok(6)];`
3. Assume an error that prevents further processing has occurred while processing `A`, but processing of `B` and `H` has already been initiated and succeeds. The not-processed elements are filled up with `null` elements up to the rightmost processed element `H`.
  * Output: `[opt #Err(#GenericBatchError(...), opt #Ok(5), null, null, null, null, null, opt #Ok(6)];`

#### Batch Query Methods

There are two different classes of query methods in terms of their API styles defined in this standard:
1. Query methods that have (at most) one response per request in the batch. For example, `icrc7_balance_of`, which receives a vector of token ids as input and each output element is the balance of the corresponding input. Those methods perfectly lend themselves for implementation with a batch API. Those queries have an analogous API style as batch update calls, with a difference in the meaning of `null` responses.
1. Query methods that may have multiple responses for an input element. An example is `icrc7_tokens_of`, which may have many response elements for an account. Those methods require pagination. Pagination is hard to combine with the batch API style and positional responses and they complicate both the API and the implementation. Thus, the guiding principle is that such methods be non-batch paginated methods, unless there is a strong reason for a deviation from this.

The class 1 of query calls above is handled with an API style that is *almost identical* to that of batch update calls as outlined above. The main and only difference is the meaning of `null` values. For update calls, a `null` response always means that processing of the corresponding request has not been initiated, e.g., after a batch error has occurred. For query calls, errors that prevent further processing of queries are not expected as queries are read operations that should not fail. For queries, `null` may be defined to have a specific meaning per query and do not have the default semantics that the corresponding request has not been processed. Queries must process the complete contiguous request sequence from index 0 up to a given request element index and may not have further response elements after that index, but must, unlike update calls, not skip processing of some elements in the returned sequence. As queries are read-only operations that don't have the numerous failure modes of updates, this should not impose any undue constraints on an implementation.

#### Error Handling

It is recommended that neither query nor update calls trap unless completely unavoidable. The API is designed such that many error cases do not need to cause a trap, but can be communicated back to the client and the processing of large batches may be short-circuited to processing only a prefix thereof in case of an error.

For example, if a limit expressed through an `icrc7:max_...` metadata attribute is violated, e.g., the maximum batch size is exceeded and the response size would exceed the system's permitted maximum, the ledger should process only a prefix of the input and return a corresponding response vector with elements corresponding to this prefix. Only a prefix of the request being responded to means that the suffix of the request has not been processed and the processing of its elements has not even been attempted to be initiated.

#### Other Aspects

The size of responses to messages sent to a canister smart contract on the IC is constrained to a fixed constant size. For requests that could potentially result in larger response messages that breach this limit, the caller SHOULD ensure to constrain the input of the methods accordingly so that the response remains below the maximum allowed size, e.g., the caller should not query too many token ids in one batch call. To avoid hitting the size limit, the ledger may process only a prefix of the request. The ledger MAY make sure that the response size does not exceed the permitted maximum *before* making any changes that might be committed to replicated state.

All update methods take `memo` parameters as input. An implementation of this standard SHOULD allow memos of at least 32 bytes in length for all methods.

Each used Candid type is only specified once in the standard text upon its first use and subsequent uses refer to this first use. Likewise, error responses may not be explained repeatedly for all methods after having been explained already upon their first use, so the reader may need to refer back to a previous use.

### icrc7_collection_metadata

Returns all the collection-level metadata of the NFT collection in a single query.

The data model for metadata is based on the generic `Value` type which allows for encoding arbitrarily complex data for each metadata attribute. The metadata attributes are expressed as `(text, value)` pairs where the first element is the name of the metadata attribute and the second element the corresponding value expressed through the `Value` type.

Analogous to [ICRC-1 metadata](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-1#metadata), metadata keys are arbitrary Unicode strings and must follow the pattern `<namespace>:<key>`, where `<namespace>` is a string not containing colons. The namespace `icrc7` is reserved for keys defined in the ICRC-7 standard.

The set of elements contained in a specific ledger's metadata depends on the ledger implementation, the list below establishes the currently defined fields.

The following metadata fields are defined by ICRC-7, starting with general collection-specific metadata fields:
  * `icrc7:symbol` of type `text`: The token symbol. Token symbols are often represented similar to [ISO-4217](https://en.wikipedia.org/wiki/ISO_4217)) currency codes. Should be the same as the result of the [`icrc7_symbol`](#icrc7_symbol) query call.
  * `icrc7:name` of type `text`: The name of the token. Should be the same as the result of the [`icrc7_name`](#icrc7_name) query call.
  * `icrc7:description` of type `text` (optional): A textual description of the token. When present, should be the same as the result of the [`icrc7_description`](#icrc7_description) query call.
  * `icrc7:logo` of type `text` (optional): The URL of the token logo. It may be a [DataURL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs) that contains the logo image itself. When present, should be the same as the result of the [`icrc7_logo`](#icrc7_logo) query call.
  * `icrc7:total_supply` of type `nat`: The current total supply of the token, i.e., the number of tokens in existence. Should be the same as the result of the [`icrc7_total_supply`](#icrc7_total_supply) query call.
  * `icrc7:supply_cap` of type `nat` (optional): The current maximum supply for the token beyond which minting new tokens is not possible. When present, should be the same as the result of the [`icrc7_supply_cap`](#icrc7_supply_cap) query call.

The following are the more technical, implementation-oriented, metadata elements:
  * `icrc7:max_query_batch_size` of type `nat` (optional): The maximum batch size for batch query calls this ledger implementation supports. When present, should be the same as the result of the [`icrc7_max_query_batch_size`](#icrc7_max_query_batch_size) query call.
  * `icrc7:max_update_batch_size` of type `nat` (optional): The maximum batch size for batch update calls this ledger implementation supports. When present, should be the same as the result of the [`icrc7_max_update_batch_size`](#icrc7_max_update_batch_size) query call.
  * `icrc7:default_take_value` of type `nat` (optional): The default value this ledger uses for the `take` pagination parameter which is used in some queries. When present, should be the same as the result of the [`icrc7_default_take_value`](#icrc7_default_take_value) query call.
  * `icrc7:max_take_value` of type `nat` (optional): The maximum `take` value for paginated query calls this ledger implementation supports. The value applies to all paginated queries the ledger exposes. When present, should be the same as the result of the [`icrc7_max_take_value`](#icrc7_max_take_value) query call.
  * `icrc7:max_memo_size` of type `nat` (optional): The maximum size of `memo`s as supported by an implementation. When present, should be the same as the result of the [`icrc7_max_memo_size`](#icrc7_max_memo_size) query call.
  * `icrc7:atomic_batch_transfers` of type `bool` (optional): `true` if and only if batch transfers of the ledger are executed atomically, i.e., either all transfers execute or none, `false` otherwise. Defaults to `false` if the attribute is not defined.
  * `icrc7:tx_window` of type `nat` (optional): The time window in seconds during which transactions can be deduplicated. Corresponds to the parameter `TX_WINDOW` as specified in the section on [transaction deduplication](#transaction_deduplication).
  * `icrc7:permitted_drift` of type `nat` (optional): The time duration in seconds by which the transaction deduplication window can be extended. Corresponds to the parameter `PERMITTED_DRIFT` as specified in the section on [transaction deduplication](#transaction_deduplication).

Note that if `icrc7_max...` limits specified through metadata are violated in a query call by providing larger argument lists or resulting in larger responses than permitted, the canister SHOULD return a response only to a prefix of the request items.

```candid "Type definitions" +=
// Generic value in accordance with ICRC-3
type Value = variant { 
    Blob : blob; 
    Text : text; 
    Nat : nat;
    Int : int;
    Array : vec Value; 
    Map : vec record { text; Value }; 
};
```

```candid "Methods" +=
icrc7_collection_metadata : () -> (vec record { text; Value } ) query;
```

### icrc7_symbol

Returns the token symbol of the NFT collection (e.g., `MS`).

```candid "Methods" +=
icrc7_symbol : () -> (text) query;
```

### icrc7_name

Returns the name of the NFT collection (e.g., `My Super NFT`).

```candid "Methods" +=
icrc7_name : () -> (text) query;
```

### icrc7_description

Returns the text description of the collection.

```candid "Methods" +=
icrc7_description : () -> (opt text) query;
```

### icrc7_logo

Returns a link to the logo of the collection. It may be a [DataURL](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URLs) that contains the logo image itself.

```candid "Methods" +=
icrc7_logo : () -> (opt text) query;
```

### icrc7_total_supply

Returns the total number of NFTs on all accounts.

```candid "Methods" +=
icrc7_total_supply : () -> (nat) query;
```

### icrc7_supply_cap

Returns the maximum number of NFTs possible for this collection. Any attempt to mint more NFTs 
than this supply cap shall be rejected.

```candid "Methods" +=
icrc7_supply_cap : () -> (opt nat) query;
```

### icrc7_max_query_batch_size

Returns the maximum batch size for batch query calls this ledger implementation supports.

```candid "Methods" +=
icrc7_max_query_batch_size : () -> (opt nat) query;
```

### icrc7_max_update_batch_size

Returns the maximum number of token ids allowed for being used as input in a batch update method.

```candid "Methods" +=
icrc7_max_update_batch_size : () -> (opt nat) query;
```

### icrc7_default_take_value

Returns the default parameter the ledger uses for `take` in case the parameter is `null` in paginated queries.

```candid "Methods" +=
icrc7_default_take_value : () -> (opt nat) query;
```

### icrc7_max_take_value

Returns the maximum `take` value for paginated query calls this ledger implementation supports. The value applies to all paginated calls the ledger exposes.

```candid "Methods" +=
icrc7_max_take_value : () -> (opt nat) query;
```

### icrc7_max_memo_size

Returns the maximum size of `memo`s as supported by an implementation.

```candid "Methods" +=
icrc7_max_memo_size : () -> (opt nat) query;
```

### icrc7_atomic_batch_transfers

Returns `true` if and only if batch transfers of the ledger are executed atomically, i.e., either all transfers execute or none, `false` otherwise.

```candid "Methods" +=
icrc7_atomic_batch_transfers : () -> (opt bool) query;
```

### icrc7_tx_window

Returns the time window in seconds during which transactions can be deduplicated. Corresponds to the parameter `TX_WINDOW` as specified in the section on [transaction deduplication](#transaction_deduplication).

```candid "Methods" +=
icrc7_tx_window : () -> (opt nat) query;
```

### icrc7_permitted_drift

Returns the time duration in seconds by which the transaction deduplication window can be extended. Corresponds to the parameter `PERMITTED_DRIFT` as specified in the section on [transaction deduplication](#transaction_deduplication).

```candid "Methods" +=
icrc7_permitted_drift : () -> (opt nat) query;
```

### icrc7_token_metadata

Returns the token metadata for `token_ids`, a list of token ids. Each tuple in the response vector comprises an optional `metadata` element with the metadata expressed as vector of `text` and `Value` pairs. In case a token does not exist, a `null` element corresponding to it is returned in the response. If a token does not have metadata, its associated metadata vector is the empty vector.

ICRC-7 does not specify the representation of token metadata any further than that it is represented in a generic manner as a vector of `(text, Value)`-pairs. This is left to future standards, the collections, the implementations, or emerging best practices, in order to not unnecessarily constrain the utility and applicability of this standard.

> [!NOTE]
> Encoding of types not contained in the `Value` type SHOULD be handled according to best practices as put forth in the context of the ICRC-3 standard.

Token metadata is expressed using the same `Value` type as used for collection metadata:
```candid
// Generic value in accordance with ICRC-3
type Value = variant { 
    Blob : blob; 
    Text : text; 
    Nat : nat;
    Int : int;
    Array : vec Value; 
    Map : vec record { text; Value }; 
};
```

```candid "Methods" +=
icrc7_token_metadata : (token_ids : vec nat)
    -> (vec opt vec record { text; Value }) query;
```

### icrc7_owner_of

Returns the owner `Account` of each token in a list `token_ids` of token ids. The ordering of the response elements corresponds to that of the request elements.

Tokens for which an ICRC-1 account cannot be found have a `null` response. This can, for example, be the case for a ledger that has originally used a different token standard, e.g., based on the ICP account model, and tokens of this ledger have not been fully migrated yet to ICRC-7. Non-existing token ids also receive a `null` response.

```candid "Methods" +=
icrc7_owner_of : (token_ids : vec nat)
    -> (vec opt Account) query;
```

### icrc7_balance_of

Returns the balance of the `account` provided as an argument, i.e., the number of tokens held by the account. For a non-existing account, the value `0` is returned.

```candid "Methods" +=
icrc7_balance_of : (vec Account) -> (vec nat) query;
```

### icrc7_tokens

Returns the list of tokens in this ledger, sorted by their token id.

The result is paginated and pagination is controlled via the `prev` and `take` parameters: The response to a request results in at most `take` many token ids, starting with the next id following `prev`. The token ids in the response are sorted in any consistent sorting order used by the ledger. If `prev` is `null`, the response elements start with the smallest ids in the ledger according to the sorting order. If the response to a call with a non-null `prev` value contains no token ids, there are no further tokens following `prev`. If the response to a call contains fewer token ids than the provided or default `take` value, there are no further tokens in the ledger following the largest returned token id. If `take` is omitted, the ledger's default `take` value as specified through `icrc7:default_take_value` is assumed.

For retrieving all tokens of the ledger, the pagination API is used such that the first call sets `prev = null` and specifies a suitable `take` value. Then, the method is called repeatedly such that the greatest token id of the previous response is used as `prev` value for the next call to the method. The method is called in this manner as long as the response comprises `take` many elements if take has been specified or `icrc7:default_take_value` many elements if `take` has not been specified. When a response comprises fewer elements than `take` or the `icrc7:default_take_value`, respectively, iterating can be stopped as the end of the token sequence has been reached. Using this approach, all tokens can be enumerated in ascending order, provided the ledger state does not change between the method calls.

Each invocation is executed on the current memory state of the ledger. I.e., it is not possible to enumerate the exact list of token ids of the ledger at a given time or of a "snapshot" of the ledger state. Rather, the ledger state can change between the multiple calls required to enumerate all the tokens.

```candid "Methods" +=
icrc7_tokens : (prev : opt nat, take : opt nat)
    -> (vec nat) query;
```

### icrc7_tokens_of

Returns a vector of `token_id`s of all tokens held by `account`, sorted by `token_id`.  The token ids in the response are sorted in any consistent sorting order used by the ledger. The result is paginated, the mechanics of pagination are analogous to `icrc7_tokens` using `prev` and `take` to control pagination.

```candid "Methods" +=
icrc7_tokens_of : (account : Account, prev : opt nat, take : opt nat)
    -> (vec nat) query;
```

### icrc7_transfer

Performs a batch of token transfers. Each of those transfers transfers a token `token_id` from the account defined by the caller principal and the specified `from_subaccount` to the `to` account. A `null` for the `from_subaccount` refers to the default subaccount comprising all zeroes. A `memo` and `created_at_time` can be given optionally. The transfer can only be initiated by the holder of the tokens.

The method response comprises a vector of optional elements, one per request element. The response is a positional argument w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the transfer in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

A transfer clears all active token-level approvals for a successfully transferred token. This implicit clearing of approvals only clears token-level approvals and never touches collection-level approvals. This clearing does not create an ICRC-3 block in the transaction log, but it is implied by the transfer block in the log.

Batch transfers are *not atomic* by default, i.e., a user SHOULD not assume that either all or none of the transfers have been executed. A ledger implementation MAY choose to implement atomic batch transfers, in which case the metadata attribute `icrc7_atomic_batch_transfers` is set to `true`. If an implementation does not specifically implement batch atomicity, batch transfers are not atomic due to the asynchronous call semantics of the Internet Computer platform. An implementor of this standard who implements atomic batch transfers and advertises those through the `icrc7_atomic_batch_transfers` metadata attribute MUST take great care to ensure everything required has been considered to achieve atomicity of the batch of transfers.

```candid "Type definitions" +=
type TransferArg = record {
    from_subaccount: opt blob; // The subaccount to transfer the token from
    to : Account;
    token_id : nat;
    memo : opt blob;
    created_at_time : opt nat64;
};

type TransferResult = variant {
    Ok : nat; // Transaction index for successful transfer
    Err : TransferError;
};

type TransferError = variant {
    NonExistingTokenId;
    InvalidRecipient;
    Unauthorized;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc7_transfer : (vec TransferArg) -> (vec opt TransferResult);
```

The ledger returns an `InvalidRecipient` error in case `to` equals `from` for a `TransferArg`.

If the caller principal is not permitted to act on a token id, then the corresponding request item receives the `Unauthorized` error response. This may be the case if the token is not held in the specified subaccount `from_subaccount`.

The `memo` parameter is an arbitrary blob that is not interpreted by the ledger. The ledger SHOULD allow memos of at least 32 bytes in length. The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction-deduplication).

The ledger SHOULD reject transactions with the `Duplicate` error variant in case the transaction is found to be a duplicate based on the [transaction deduplication](#transaction-deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction.
The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

> [!NOTE]
> Note that multiple concurrently executing batch transfers triggered by method invocations can lead to an interleaved execution of the corresponding sequences of token transfers.

> [!NOTE]
> Note further that deduplication is performed independently on the different items of the batch.

### icrc10_supported_standards

An implementation of ICRC-7 MUST implement the method `icrc10_supported_standards` as put forth in ICRC-10.

The result of the call MUST always have at least the following entries:

```candid
record { name = "ICRC-7"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-7"; }
record { name = "ICRC-10"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-10"; }
```

## ICRC-7 Block Schema

ICRC-7 builds on the [ICRC-3](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3) specification for defining the format for storing transactions in blocks of the log of the ledger. ICRC-3 defines a generic, extensible, block schema that can be further instantiated in standards implementing ICRC-3. We next define the concrete block schema for ICRC-7 as extension of the ICRC-3 block schema. This schema must be implemented by a ledger implementing ICRC-7 if it claims to implement ICRC-3 through the method listing the supported standards.

### Generic ICRC-7 Block Schema

An ICRC-7 block is defined as follows:
1. its `btype` field MUST be set to the op name that starts with `7`
2. it MUST contain a field `ts: Nat` which is the timestamp of when the block was added to the Ledger
3. it MUST contain a field `tx`, which
    1. MAY contain a field `memo: Blob` if specified by the user
    2. MAY contain a field `ts: Nat` if the user sets the `created_at_time` field in the request.

The `tx` field contains the transaction data as provided by the caller and is further refined for each the different update calls as specified below.

### Mint Block Schema

1. the `btype` field of the block MUST be set to `"7mint"`
2. the `tx` field
    1. MUST contain a field `tid: Nat`
    2. MAY contain a field `from: Account`
    3. MUST contain a field `to: Account`
    4. MUST contain a field `meta: Value`

Note that `tid` refers to the token id. The size of the `meta` field expressing the token metadata must be small enough such that the block fits the size limit for inter-canister calls. The `meta` field SHOULD, if no extension standard is used that defines a different means of expressing the metadata, contain a `Map` variant of `Value` with the single entry `("icrc7:token_metadata", metadata)`, where `metadata` is the actual metadata of the token expressed in a `Map` variant of `Value`. This approach of including the full metadata guarantees that the ledger state can be completely reproduced from the block log.

### Burn Block Schema

1. the `btype` field of the block MUST be set to `"7burn"`
2. the `tx` field
    1. MUST contain a field `tid: Nat`
    2. MUST contain a field `from: Account`
    3. MAY contain a field `to: Account`

### icrc7_transfer Block Schema

1. the `btype` field of the block MUST be set to `"7xfer"`
2. the `tx` field
    1. MUST contain a field `tid: Nat`
    2. MUST contain a field `from: Account`
    3. MUST contain a field `to: Account`

As `icrc7_transfer` is a batch method, it results in one block per `token_id` in the batch. The method results in one block per input of the batch. The blocks need not appear in the block log in the same relative sequence as the token ids appear in the vector of input token identifiers in order to not unnecessarily constrain the potential concurrency of an implementation. The block sequence corresponding to the token ids in the input can be interspersed with blocks from other (batch) methods executed by the ledger in an interleaved execution sequence. This allows for high-performance ledger implementations that can make asynchronous calls to other canisters in the scope of operations on tokens and process multiple batch update calls concurrently.

### Update Token Block Schema

1. the `btype` field of the block MUST be set to `"7update_token"`
2. the `tx` field
    1. MUST contain a field `tid: Nat`
    2. MAY contain a field `from: Account` with an account that initiated the update
    3. MUST contain a field `meta: Value` with the metadata or metadata hash

Analogous to the mint block schema, `meta` SHOULD, if no extension standard is used that defines a different means of expressing the metadata, contain a `Map` variant of `Value` with the single entry `("icrc7:token_metadata", metadata)`, where `metadata` is the actual updated metadata of the token expressed in a `Map` variant of `Value`. This approach of including the full udpated metadata guarantees that the ledger state can be completely reproduced from the block log.

Note that there is no method defined in this specification for the metadata update, but this is left to the implementation of the ledger or a future standard.

Future extension standards can define more storage-efficient mechanisms for storing only deltas in update blocks as well as mechanisms for storing metadata outside the NFT ledger.

## Migration Path for Ledgers Using ICP AccountId

For historical reasons, multiple NFT standards, such as the EXT standard, use the ICP `AccountIdentifier` or `AccountId` (a hash of the principal and subaccount) instead of the ICRC-1 `Account` (a pair of principal and subaccount) to store the owners. Since the ICP `AccountId` can be calculated from an ICRC-1 `Account`, but computability does not hold in the inverse direction, there is no way for a ledger implementing ICP `AccountId` to display `icrc7_owner_of` data.

This standard does not mandate any provisions regarding the handling of tokens managed through the `AccountId` regime and leaves this open to a future ICRC standard that ledgers may opt to implement or ledger-specific implementations.

Ledgers using the ICP `AccountId` can provide a `null` response for a token that has not yet been migrated to an ICRC-1 account for the `icrc7_owner_of` query. The ledger implementation may offer an additional method to allow clients to obtain further information on this token, e.g., whether it is a token based on the ICP `AccountId`. `AccountId`-based ledgers that want to support ICRC-7 need to implement a strategy to become ICRC-7 compliant, e.g., by requiring all users to call a migration endpoint to migrate their tokens to an ICRC-1-based representation. It is acceptable behaviour to not consider not-yet-migrated tokens of such ledgers in responses as they conceptually don't count against the total ICRC-7 supply of the ledger before being migrated.

Different approaches for migration are feasible and the choice of migration approach is left to the ledger implementation and not mandated in this standard. ICRC standards may emerge in the future for addressing the migration from previous NFT standards to ICRC-7.

## Extensions

The base standard intentionally excludes some ledger functions essential for building a rich DeFi ecosystem, for example:

  * Reliable transaction notifications for smart contracts.
  * The block structure and the interface for fetching blocks.
  * Pre-signed transactions.

The standard uses the `icrc10_supported_standards` endpoint to accommodate these and other future extensions.
This endpoint returns names of all specifications (e.g., `"ICRC-3"` or `"ICRC-10"`) implemented by the ledger as well as URLs.

## Transaction Deduplication

Consider the following scenario:

  1. An agent sends a transaction to an ICRC-7 ledger hosted on the IC.
  2. The ledger accepts the transaction.
  3. The agent loses the network connection for several minutes and cannot learn about the outcome of the transaction.

An ICRC-7 ledger SHOULD implement transfer deduplication to simplify the error recovery for agents.
The deduplication covers all transactions submitted within a pre-configured time window `TX_WINDOW` (for example, last 24 hours).
The ledger MAY extend the deduplication window into the future by the `PERMITTED_DRIFT` parameter (for example, 2 minutes) to account for the time drift between the client and the Internet Computer.

The client can control the deduplication algorithm using the `created_at_time` and `memo` fields of the [`transfer`](#icrc7_transfer) call argument:
  * The `created_at_time` field sets the transaction construction time as the number of nanoseconds from the UNIX epoch in the UTC timezone.
  * The `memo` field does not have any meaning to the ledger, except that the ledger will not deduplicate transfers with different values of the `memo` field.

The ledger SHOULD use the following algorithm for transaction deduplication if the client has set the `created_at_time` field:
  * If `created_at_time` is set and is _before_ `time() - TX_WINDOW - PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { TooOld }` error.
  * If `created_at_time` is set and is _after_ `time() + PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { CreatedInFuture = record { ledger_time = ... } }` error.
  * If the ledger observed a structurally equal transfer payload (i.e., all the transfer argument fields and the caller have the same values) at transaction with index `i`, it should return `variant { Duplicate = record { duplicate_of = i } }`.
  * Otherwise, the transfer is a new transaction.

If the client has not set the `created_at_time` field, the ledger SHOULD NOT deduplicate the transaction.

## Security Considerations

This section highlights some selected areas crucial for security regarding the implementation of ledgers following this standard and Web applications using ledgers following this standard. Note that this is not exhaustive by any means, but rather points out a few selected important areas.

### Protection Against Denial of Service Attacks

It is strongly recommended that implementations of this standard take steps towards protecting against Denial of Service (DoS) attacks. Some non-exhaustive list of examples for recommended mitigations are given next:
  * Enforcing limits, such as the number of active approvals per token for token-level approvals or per principal for collection-level approvals, to constrain the state size of the ledger. Examples of such limits are given in this standard through various metadata attributes.
  * Enforcing rate limits, such as the number of transactions such as approvals or approval revocations, can be performed on a per-token and per-principal basis to constrain the size of the transaction log for the ledger.
  * The execution of operations such as approving collections and revoking such approvals could be constrained to parties who own at least one token on a ledger. This helps prevent DoS by attackers who create a large number of principals and perform such operations without holding tokens.

### Protection Against Web Application Attacks

We strongly advise developers who display untrusted user-generated data like images (e.g., the token logo or images referenced from NFT metadata) or strings in a Web application to follow Web application security best practices to avoid attacks such as XSS and CSRF resulting from malicious content provided by a ledger. As one particular example, images in the SVG format provide potential for attacks if used improperly. See, for example, the OWASP guidelines for protecting against [XSS](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) or [CSRF](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html). The above examples are only selected examples and by no means provide an exhaustive view of security issues.

<!--
```candid ICRC-7.did +=
<<<Type definitions>>>

service : {
  <<<Methods>>>
}
```
-->

# Metadata

The following metadata properties are defined for ICRC-37:
  * `icrc37:max_approvals_per_token_or_collection` of type `nat` (optional): The maximum number of active approvals this ledger implementation allows per token or per principal for the collection. When present, should be the same as the result of the [`icrc37_max_approvals_per_token_or_collection`](#icrc37_max_approvals_per_token_or_collection) query call.
  * `icrc37:max_revoke_approvals` of type `nat` (optional): The maximum number of approvals that may be revoked in a single invocation of `icrc37_revoke_token_approvals` or `icrc37_revoke_collection_approvals`. When present, should be the same as the result of the [`icrc37_max_revoke_approvals`](#icrc37_max_revoke_approvals) query call.

Those metadata attributes can be obtained through the `icrc7_collection_metadata` method of the ledger. All other relevant `icrc7:...` metadata properties from the ICRC-7 implementation that implements this standard apply to this standard, e.g., the maximum batch sizes for queries and updates or the transaction deduplication parameters.

## Concepts

*Approvals* allow a principal, the *spender*, to transfer tokens owned by another account that has approved the spender, where the transfer is performed on behalf of the owner. Approvals can be created on a per-token basis using `icrc37_approve_tokens` or for the whole collection, i.e., for all tokens of the collection, using `icrc37_approve_collection`. The owner principal can explicitly revoke an active approval at their discretion using the `icrc37_revoke_token_approvals` for revoking token-level approvals and `icrc37_revoke_collection_approvals` for revoking collection-level approvals. A transfer of a token implicitly revokes all token-level approvals of the transferred token. A collection-level approval is not affected by any changes of token ownerships and is not related to specific tokens. An approval that has been created, has not expired (i.e., the `expires_at` field is a date in the future), has not been revoked implicitly through a transfer of the approved token, has not been revoked explicitly, and has not been replaced with a new approval is *active*, i.e., can allow the approved party to initiate a transfer.

When an active approval exists for a token or for an account for the whole collection, the spender specified in the approval can transfer tokens within the scope of the approval using the `transfer_from` method. A successful transfer implicitly revokes all active token-level approvals for the token in addition to performing the actual token transfer. Collection-level approvals are never revoked by transfers as they are not related to specific tokens, but the collection as a whole.

Analogous to ICRC-7, also ICRC-37 uses the ICRC-1 *account* as entity that the source account (`from`), destination account (`to`), and spending account (`spender`) are expressed with, i.e., a *subaccount* is always used besides the principal. We follow the naming convention of using `from_subaccount` for subaccounts being part of the source account and `spender_subaccount` for subaccounts being part the spender account. In many practical situations the default subaccount, i.e., the all-`0` subaccount, is expected to be used.

## Data Representation

### Accounts

Accounts and subaccounts are defined as in ICRC-37. For convenience, the core ideas are reiterated here.
A `principal` can have multiple accounts. Each account of a `principal` is identified by a 32-byte string called `subaccount`. Therefore, an account corresponds to a pair `(principal, subaccount)`.

The account identified by the subaccount with all bytes set to 0 is the *default account* of the `principal`.

```candid "Type definitions" +=
type Subaccount = blob;
type Account = record { owner : principal; subaccount : opt Subaccount };
```

See [ICRC-7](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md#accounts) for further details on the concept of accounts and subaccounts.

## Methods

### Generally-Applicable Specification

The general principles on the API as put forth in [ICRC-7](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md) apply also to ICRC-37.

To summarize, all eligible calls, i.e., such with (at most) one response element per request, are batch calls. Batch calls have *positional responses*, i.e., the `i`-th response element is the response to the `i`-th request element. The response may contain responses only to a prefix of the request. For update calls, responses can contain `null` elements, with the meaning that processing of the corresponding request has not been initiated. For query calls, `null` responses have a meaning as determined by the respective query call.

### icrc37_max_approvals_per_token_or_collection

Returns the maximum number of approvals this ledger implementation allows to be active per token or per principal for the collection.

```candid "Methods" +=
icrc37_max_approvals_per_token_or_collection : () -> (opt nat) query;
```

### icrc37_max_revoke_approvals

Returns the maximum number of approvals that may be revoked in a single invocation of `icrc37_revoke_token_approvals` or `icrc37_revoke_collection_approvals`.

```candid "Methods" +=
icrc37_max_revoke_approvals : () -> (opt nat) query;
```

### icrc37_approve_tokens

Entitles a `spender`, specified through an `Account`, to transfer NFTs on behalf of the caller of this method from `account { owner = caller; subaccount = from_subaccount }`, where `caller` is the caller of this method (and also the owner principal of the tokens that are subject to approval) and `from_subaccount` is the subaccount of the token owner principal the approval should apply to (i.e., the subaccount which the tokens must be held on and can be transferred out from). The `from_subaccount` being `null` refers to the default subaccount. Note that the `from_subaccount` parameter needs to be specified because accounts are a primary concept in this standard and thereby the `from_subaccount` needs to be specified as part of the account that holds the token. The `expires_at` value specifies the expiration date of the approval. The method has batch semantics and allows for submitting a batch of such token-level approvals with a single invocation.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the token-level approval in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

Only one approval can be active for a given `(token_id, spender)` pair (the `from_subaccount` of the approval must be equal to the subaccount the token is held on).

Multiple approvals can exist for the same `token_id` but different `spender`s (the `from_subaccount` field must be the same and equal to the subaccount the token is held on). In case an approval for the specified `spender` already exists for a token on `from_subaccount` of the caller, a new approval is created that replaces the existing approval. The replaced approval is superseded with the effect that the new parameters for the approval (`expires_at`, `memo`, `created_at_time`) apply. The ledger SHOULD limit the number of approvals that can be active per token to constrain unlimited growth of ledger memory. Such limit is exposed as ledger metadata through the metadata attribute `icrc37:max_approvals_per_token_or_collection`.

An ICRC-7 ledger implementation does not need to keep track of expired approvals in its memory. This is important to help constrain unlimited growth of ledger memory over time. All historic approvals are contained in the block history the ledger creates.

The ledger returns an `InvalidSpender` error if the spender account owner is equal to the caller account owner. I.e., a principal cannot create an approval for themselves, because a principal always has an implicit approval to act on their own tokens.

An `Unauthorized` error is returned in case the caller is not authorized to perform this action on the token, i.e., it does not own the token or the token is not held in the account specified through `from_subaccount`.

A `NonExistingTokenId` error is returned in case the referred-to token does not exist.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction, the `memo` parameter is an arbitrary blob that is not interpreted by the ledger. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

```candid "Type definitions" +=
type ApprovalInfo = record {
    spender : Account;             // Approval is given to an ICRC Account
    from_subaccount : opt blob;    // The subaccount the token can be transferred out from with the approval
    expires_at : opt nat64;
    memo : opt blob;
    created_at_time : nat64; 
};

type ApproveTokenArg = record {
    token_id : nat;
    approval_info : ApprovalInfo;
};

type ApproveTokenResult = variant {
    Ok : nat; // Transaction index for successful approval
    Err : ApproveTokenError;
};

type ApproveTokenError = variant {
    InvalidSpender;
    Unauthorized;
    NonExistingTokenId;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_approve_tokens : (vec ApproveTokenArg)
    -> (vec opt ApproveTokenResult);
```

### icrc37_approve_collection

Entitles a `spender`, specified through an `Account`, to transfer any NFT of the collection hosted on this ledger and owned by the caller at the time of transfer on behalf of the caller of this method from `account { owner = caller; subaccount = from_subaccount }`, where `caller` is the caller of this method and `from_subaccount` is the subaccount of the token owner principal the approval should apply to (i.e., the subaccount which tokens the approval should apply to must be held on and can be transferred out from). The `from_subaccount` being `null` refers to the default subaccount. Note that the `from_subaccount` parameter needs to be specified not only because accounts are a primary concept in this standard, but also because the approval applies to the collection, i.e., all tokens on the ledger the caller holds, and those tokens may be held on different subaccounts. The `expires_at` value specifies the expiration date of the approval. The method has batch semantics and allows for submitting a batch of such collection approvals with a single invocation.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the collection-level approval in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

Only one approval can be active for a given `(spender, from_subaccount)` pair. Note that it is not required that tokens be held by the caller on their `from_subaccount` for the approval to be active.

Multiple approvals can exist for the collection for a caller but different `spender`s and `from_subaccount`s, i.e., one approval per `(spender, from_subaccount)` pair. In case an approval for the specified `spender` and `from_subaccount` of the caller for the collection already exists, a new approval is created that replaces the existing approval. The replaced approval is superseded with the effect that the new parameters (`expires_at`, `memo`, `created_at_time`) apply to the approval defined by `from_subaccount` and `spender`. The ledger SHOULD limit the number of approvals that can be active per collection to constrain unlimited growth of ledger memory. Such limit is exposed as ledger metadata through the metadata attribute `icrc37:max_approvals_per_token_or_collection`.

An ICRC-7 ledger implementation does not need to keep track of expired approvals in its memory. This is important to help constrain unlimited growth of ledger memory over time. All historic approvals are contained in the block log history the ledger creates.

It is left to the ledger implementation to decide whether collection-level approvals can be successfully created independently of currently owning tokens of the collection at approval time.

The ledger returns an `InvalidSpender` error if the spender account owner is equal to the caller account owner. I.e., a principal cannot create an approval for themselves, because a principal always has an implicit approval to act on their own tokens.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction, the `memo` parameter is an arbitrary blob that is not interpreted by the ledger. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

Note that this method is analogous to `icrc37_approve_tokens`, but for approving whole collections. `ApproveCollectionArg` specifies the approval to be made for the collection.

To ensure proper semantics, collection-level approvals MUST be managed by the ledger as collection-level approvals and MUST NOT be translated into token-level approvals for all tokens the caller currently owns.

See the [#icrc37_approve_tokens](#icrc37_approve_tokens) method for the `ApprovalInfo` type.

```candid "Type definitions" +=
type ApproveCollectionArg = record {
    approval_info : ApprovalInfo;
};

type ApproveCollectionResult = variant {
    Ok : nat; // Transaction index for successful approval
    Err : ApproveCollectionError;
};

type ApproveCollectionError = variant {
    InvalidSpender;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_approve_collection : (vec ApproveCollectionArg)
    -> (vec opt ApproveCollectionError);
```

### icrc37_revoke_token_approvals

Revokes the specified approvals for a token given by `token_id` from the set of active approvals. The `from_subaccount` parameter specifies the token owner's subaccount to which the approval applies, the `spender` the party for which the approval is to be revoked. A `null` for the spender means to revoke approvals that match the request for all spenders. A `null` value of `from_subaccount` indicates the default subaccount. The method allows for a batch of token approval revocations in a single invocation.

Only the owner of tokens can revoke approvals through this method.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the token-level approval revocation in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

An `ApprovalDoesNotExist` error is returned in case the approval to be revoked is not an existing active approval.

An `Unauthorized` error is returned in case the caller is not authorized to perform this action on the token.

A `NonExistingTokenId` error is returned in case the referred-to token does not exist.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

Revoking an approval for one or more token ids does not affect collection-level approvals.

An ICRC-37 ledger implementation does not need to keep track of revoked approvals in memory. Revoked approvals are always available in the transaction log.

```candid "Type definitions" +=
type RevokeTokenApprovalArg = record {
    spender : opt Account;      // null revokes matching approvals for all spenders
    from_subaccount : opt blob; // null refers to the default subaccount
    token_id : nat;
    memo : opt blob;
    created_at_time : opt nat64;
};

type RevokeTokenApprovalResponse = variant {
    Ok : nat; // Transaction index for successful approval revocation
    Err : RevokeTokenApprovalError;
};

type RevokeTokenApprovalError = variant {
    ApprovalDoesNotExist;
    Unauthorized;
    NonExistingTokenId;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_revoke_token_approvals: (vec RevokeTokenApprovalArg)
    -> (vec opt RevokeTokenApprovalResponse);
```

### icrc37_revoke_collection_approvals

Revokes collection-level approvals from the set of active approvals. The `from_subaccount` parameter specifies the token owner's subaccount to which the approval applies, the `spender` the party for which the approval is to be revoked. A `null` for the spender means to revoke approvals that match the request for all spenders. A `null` value of `from_subaccount` indicates the default subaccount. The method allows for a batch of token approval revocations in a single invocation.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the collection-level approval revocation in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

This is the analogous method to `icrc37_revoke_token_approvals` for revoking collection-level approvals.

Revoking a collection-level approval does not affect token-level approvals for individual token ids.

An `ApprovalDoesNotExist` error is returned in case the approval to be revoked is not an existing active approval.

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

An ICRC-37 ledger implementation does not need to keep track of revoked approvals in memory. Revoked approvals are always available in the transaction log.

```candid "Type definitions" +=
type RevokeCollectionApprovalArg = record {
    spender : opt Account;      // null revokes approvals for all spenders that match the remaining parameters
    from_subaccount : opt blob; // null refers to the default subaccount
    memo : opt blob;
    created_at_time : opt nat64;
};

type RevokeCollectionApprovalResult = variant {
    Ok : nat; // Transaction index for successful approval revocation
    Err : RevokeCollectionApprovalError;
};

type RevokeCollectionApprovalError = variant {
    ApprovalDoesNotExist;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_revoke_collection_approvals: (vec RevokeCollectionApprovalArg)
    -> (vec opt RevokeCollectionApprovalResult);
```

### icrc37_is_approved

Returns `true` if an active approval, i.e., a token-level approval or collection-level approval, exists that allows the `spender` to transfer the token `token_id` from the given `from_subaccount`, `false` otherwise.

```candid "Type definitions" +=
type IsApprovedArg = record {
    spender : Account;
    from_subaccount : opt blob;
    token_id : nat;
};
```

```candid "Methods" +=
icrc37_is_approved : (vec IsApprovedArg)
    -> (vec bool) query;
```

### icrc37_get_token_approvals

Returns the token-level approvals that exist for the given `token_id`.  The result is paginated, the mechanics of pagination are analogous to `icrc7_tokens` using `prev` and `take` to control pagination, with `prev` being of type `TokenApproval`. Note that `take` refers to the number of returned elements to be requested. The `prev` parameter is a `TokenApproval` element with the meaning that `TokenApproval`s following the provided one are returned, based on a sorting order over `TokenApproval`s implemented by the ledger.

The response is a vector of `TokenApproval` elements. If multiple approvals exist for a token id, multiple entries of type `TokenApproval` with the same token id are contained in the response.

The ordering of the elements in the response is follows an internal ordering used by the ledger. An implementation of the ledger can use any internal sorting order for the elements of the response to implement pagination.

See the [#icrc37_approve_tokens](#icrc37_approve_tokens) for the `ApprovalInfo` type.

```candid "Type definitions" +=
type TokenApproval = record {
    token_id : nat;
    approval_info : ApprovalInfo;
};
```

```candid "Methods" +=
icrc37_get_token_approvals : (token_id : nat, prev : opt TokenApproval, take : opt nat)
    -> (vec TokenApproval) query;
```

### icrc37_get_collection_approvals

Returns the collection-level approvals that exist for the specified `owner`. The result is paginated, the mechanics of pagination are analogous to `icrc7_tokens` using `prev` and `take` to control pagination. The `prev` parameter is a `CollectionApproval` with the meaning that `CollectionApproval`s following the provided one are returned, based on a sorting order over `CollectionApproval`s implemented by the ledger.

The response is a vector of `CollectionApproval` elements.

The elements in the response are ordered following a sorting order defined by the implementation. An implementation of the ledger can use any suitable sorting order for the elements of the response to implement pagination.

See the [#icrc37_approve_tokens](#icrc37_approve_tokens) for the `ApprovalInfo` type.

```candid "Type definitions" +=
type CollectionApproval = ApprovalInfo;
```

```candid "Methods" +=
icrc37_get_collection_approvals : (owner : Account, prev : opt CollectionApproval, take : opt nat)
    -> (vec CollectionApproval) query;
```

### icrc37_transfer_from

Transfers one or more tokens from the `from` account to the `to` account. The transfer can be initiated by the holder of the tokens (the holder has an implicit approval for acting on all their tokens on their own behalf) or a party that has been authorized by the holder to execute transfers using `icrc37_approve_tokens` or `icrc37_approve_collection`. The `spender_subaccount` is used to identify the spender: The spender is an account comprised of the principal calling this method and the parameter `spender_subaccount`. Omitting the `spender_subaccount` means using the default subaccount.

The method response comprises a vector of optional elements, one per request element. The response is positional w.r.t. the request, i.e., the `i`-th response element is the response to the `i`-th request element. Each response item contains either an `Ok` variant containing the transaction index of the transfer in the success case or an `Err` variant in the error case. A `null` element in the response indicates that the corresponding request element has not been processed.

The ledger returns an `InvalidRecipient` error in case `to` equals `from`.

Each of the successful transfers in the batch implicitly clears all active token-level approvals for the successfully transferred token. This implicit clearing of approvals only clears token-level approvals and never touches collection-level approvals. No explicit entry in the block log is created for the clearing, but it is implied by the transfer entry.

Batch transfers are not atomic by default, i.e., a user SHOULD not assume that either all or none of the transfers have been executed. A ledger implementation MAY choose to implement atomic batch transfers, in which case the metadata attribute `icrc7_atomic_batch_transfers` is set to `true`. If an implementation does not specifically implement batch atomicity, batch transfers are not atomic due to the asynchronous call semantics of the Internet Computer platform. An implementor of this standard who implements atomic batch transfers and advertises those through the `icrc7_atomic_batch_transfers` metadata attribute MUST take great care to ensure everything required has been considered to achieve atomicity of the batch of transfers.

```candid "Type definitions" +=
type TransferFromArg = record {
    spender_subaccount: opt blob; // The subaccount of the caller (used to identify the spender)
    from : Account;
    to : Account;
    token_id : nat;
    // type: leave open for now
    memo : opt blob;
    created_at_time : opt nat64;
};

type TransferFromResult = variant {
    Ok : nat; // Transaction index for successful transfer
    Err : TransferFromError;
};

type TransferFromError = variant {
    InvalidRecipient;
    Unauthorized;
    NonExistingTokenId;
    TooOld;
    CreatedInFuture : record { ledger_time: nat64 };
    Duplicate : record { duplicate_of : nat };
    GenericError : record { error_code : nat; message : text };
    GenericBatchError : record { error_code : nat; message : text };
};
```

```candid "Methods" +=
icrc37_transfer_from : (vec TransferFromArg)
    -> (vec opt TransferFromResult);
```

If the caller principal is not permitted to act on a token id, then the token id receives the `Unauthorized` error response. This is the case if someone not owning a token and not being the spender in an active token-level or collection-level approval attempts to transfer a token or the token is not held in the subaccount specified in the `from` account.

The `memo` parameter is an arbitrary blob that is not interpreted by the ledger.
The ledger SHOULD allow memos of at least 32 bytes in length.
The ledger SHOULD use the `memo` argument for [transaction deduplication](#transaction-deduplication).

The ledger SHOULD reject transactions with the `Duplicate` error variant in case the transaction is found to be a duplicate based on the [transaction deduplication](#transaction-deduplication).

The `created_at_time` parameter indicates the time (as nanoseconds since the UNIX epoch in the UTC timezone) at which the client constructed the transaction. The ledger SHOULD reject transactions that have the `created_at_time` argument too far in the past or the future, returning `variant { TooOld }` and `variant { CreatedInFuture = record { ledger_time = ... } }` errors correspondingly.

## Transaction Deduplication

Consider the following scenario:

  1. An agent sends a transaction to an ICRC-37 ledger hosted on the IC.
  2. The ledger accepts the transaction.
  3. The agent loses the network connection for several minutes and cannot learn about the outcome of the transaction.

An ICRC-37 ledger SHOULD implement transfer deduplication to simplify the error recovery for agents.
The deduplication covers all transactions submitted within a pre-configured time window `TX_WINDOW` (for example, last 24 hours).
The ledger MAY extend the deduplication window into the future by the `PERMITTED_DRIFT` parameter (for example, 2 minutes) to account for the time drift between the client and the Internet Computer.

The client can control the deduplication algorithm using the `created_at_time` and `memo` fields of the [`transfer_from`](#icrc37_transfer_from) call argument:
  * The `created_at_time` field sets the transaction construction time as the number of nanoseconds from the UNIX epoch in the UTC timezone.
  * The `memo` field does not have any meaning to the ledger, except that the ledger will not deduplicate transfers with different values of the `memo` field.

The ledger SHOULD use the following algorithm for transaction deduplication if the client has set the `created_at_time` field:
  * If `created_at_time` is set and is _before_ `time() - TX_WINDOW - PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { TooOld }` error.
  * If `created_at_time` is set and is _after_ `time() + PERMITTED_DRIFT` as observed by the ledger, the ledger should return `variant { CreatedInFuture = record { ledger_time = ... } }` error.
  * If the ledger observed a structurally equal transfer payload (i.e., all the transfer argument fields and the caller have the same values) at transaction with index `i`, it should return `variant { Duplicate = record { duplicate_of = i } }`.
  * Otherwise, the transfer is a new transaction.

If the client has not set the `created_at_time` field, the ledger SHOULD NOT deduplicate the transaction.

### ICRC-37 Block Schema

ICRC-37 builds on the [ICRC-3](https://github.com/dfinity/ICRC-1/tree/main/standards/ICRC-3) specification for defining the format for storing transactions in blocks of the log of the ledger. ICRC-3 defines a generic, extensible, block schema that can be further instantiated in standards implementing ICRC-3. We next define the concrete block schema for ICRC-37 as extension of the ICRC-3 block schema.

All ICRC-37 batch methods result in one block per input of the batch. The blocks need not appear in the block log in the same relative sequence as the token ids appear in the vector of input token identifiers in order to not unnecessarily constrain the potential concurrency of an implementation. The block sequence corresponding to the token ids in the input can be interspersed with blocks from other (batch) methods executed by the ledger in an interleaved execution sequence. This allows for high-performance ledger implementations that can make asynchronous calls to other canisters in the scope of operations on tokens and process multiple batch update calls concurrently.

#### Generic ICRC-37 Block Schema

The following generic schema extends the generic schema of ICRC-3 with ICRC-37-specific elements and applies to all block defintions for blocks generated by ICRC-37. This schema must be implemented by a ledger implementing ICRC-37 if it claims to implement ICRC-3 through the method listing the supported standards.

An ICRC-37 block is defined as follows:
1. its `btype` field MUST be set to the op name that starts with `37`
2. it MUST contain a field `ts: Nat` which is the timestamp of when the block was added to the Ledger
4. it MUST contain a field `tx`, which
    1. MAY contain a field `memo: Blob` if specified by the user
    2. MAY contain a field `ts: Nat` if the user sets the `created_at_time` field in the request.

The `tx` field contains the transaction data as provided by the caller and is further refined for each the different update calls as specified below.

#### icrc37_approve_tokens Block Schema

1. the `btype` field of the block MUST be set to `"37approve"`
2. the `tx` field
    2. MUST contain a field `tid: Nat`
    3. MUST contain a field `from: Account`
    4. MUST contain a field `spender: Account`
    5. MAY contain a field `exp: Nat` if set by the user

Note that `tid` refers to the token id and `exp` to the expiry time of the approval. The names of the other fiels should speak for themselves.

#### icrc37_approve_collection Block Schema

1. the `btype` field of the block MUST be set to `"37approve_coll"`
2. the `tx` field
    1. MUST contain a field `from: Account`
    2. MUST contain a field `spender: Account`
    3. MAY contain a field `exp: Nat` if set by the user

#### icrc37_revoke_token_approvals Block Schema

1. the `btype` field of the block MUST be set to `"37revoke"`
2. the `tx` field
    1. MUST contain a field `tid: Nat`
    2. MUST contain a field `from: Account`
    3. MAY contain a field `spender: Account`

If the field `spender` is present, only the one token-level approval for this `spender`, `tid` (token id) and `from` is revoked, if the field `spender` is omitted, token-level approvals are revoked for all spenders for which approvals exist as specified by the remaining parameters `tid` (token id) and `from`. This helps reduce the log storage space required for handling explicit token-level revocations.

#### icrc37_revoke_collection_approvals Block Schema

1. the `btype` field of the block MUST be set to `"37revoke_coll"`
2. the `tx` field
    1. MUST contain a field `tx.from: Account`
    2. MAY contain a field `tx.spender: Account`

If the field `spender` is present, only the one collection-level approval for this `spender` and the given `from` value is revoked, if the field `spender` is omitted, collection-level approvals are revoked for all spenders for which approvals exist as specified by the remaining parameter `from`. This helps reduce the log storage space required for handling collection-level revocations.

#### icrc37_transfer_from Block Schema

1. the `btype` field of the block MUST be set to `"37xfer"`
2. the `tx` field
    1. MUST contain a field `tx.tid: Nat`
    2. MUST contain a field `tx.spender: Account`
    3. MUST contain a field `tx.from: Account`
    4. MUST contain a field `tx.to: Account`

## Extensions

If extension standards are used in the context of ICRC-37, those are listed with the according `icrc10_supported_standards` method of the base standard.

Conceptually, it would seem a good idea to expose a separate `supported_standards` method in ICRC-37 and list all ICRC-37 extensions with this method. However, it has been decided to expose only a single such method per ledger implementation, i.e., the `icrc7_supported_standards` method. The rationale behind not exposing a separate `icrc37_supported_standards` method that covers extensions of ICRC-37 is the following: In case of doing so, the caller would need to invoke multiple, in the general case a tree of, `supported_standards` methods, one per supported standard listed in the base method, in order to get the complete view of standards implemented by a given leder. By subsuming all supported standards in the base standard, the client can obtain this information with a single call. In most practical cases, the number of supported standards is expected to be easily manageable with this simpler approach.

<!--
```candid ICRC-37.did +=
<<<Type definitions>>>

service : {
  <<<Methods>>>
}
```
-->