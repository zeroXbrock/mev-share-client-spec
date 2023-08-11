# mev-share client lib spec

*Technical requirements for implementing a MEV-Share client library.*

---

Please note: because this specification targets multiple (very) different programming languages, we write this document using terminology that mostly borrows from object-oriented programming languages using camelCase. This is simply for ease of understanding, and does not imply that you must use object-oriented programming constructs in your library (or camelCase). While we sometimes include specific details about names or how to instantiate things, these details are not prescriptive. We encourage you to use the idiomatic style and approach that is most appropriate for your language.

# project spec

Create a client library to interact with MEV-Share via its JSON-RPC 

- must be prepared to be distributed on the language’s standard package manager
    - Rust: crates.io
    - Python: PyPi
    - Go: pkg.go.dev
- must include a linter
- must run GitHub Actions on pull request to main branch
    - lint code (use common idiomatic rules)
    - build library (release mode if applicable to your language)
- must include script to release new versions idiomatically
    - bump version number in package according to Semver rules
    - tag release in GitHub
- must include a [README.md](http://README.md) file in the project root with:
    - instructions on how to instantiate the client
    - examples of listening to events and sending a bundle
    - each public method documented with params and return type

# client spec

Refer to the official [MEV-Share Specification](https://github.com/flashbots/mev-share) for JSON-RPC implementation details.

## client definition

A mev-share client communicates with the following APIs:

- Bundle API (relay.flashbots.net)
    - accepts JSON-RPC requests over https implemented in the [JSON-RPC spec](https://docs.flashbots.net/flashbots-auction/searchers/advanced/rpc-endpoint)
    - includes functions to call `mev_sendBundle` and `mev_simBundle`.
- Event API (mev-share.flashbots.net)
    - accepts GET requests over https for the Event Stream
    - The Event Stream (the “MEV-Share mempool”) is an SSE stream served over https (https://mev-share.flashbots.net/)
    - Historical event data is served on the same endpoint, but with an `/api/v1` suffix
        - https://mev-share.flashbots.net/api/v1/history
        - https://mev-share.flashbots.net/api/v1/history/info

Also see the [Event Stream docs](https://docs.flashbots.net/flashbots-mev-share/searchers/event-stream#quickstart) for more information.

Note: these links should be hard-coded for convenience, but not required for the user to use the library; users should be able to pick between defaults and custom connection parameters.

### note on abstraction

We are experimenting with communication protocols other than JSON-RPC and SSE for the Bundle API and Event Stream, respectively, so make sure to include a layer of abstraction for calls to each API.

---

From the main entrypoint of the library (using javascript as an example: the index file), expose the following functions:

## constructor

In the constructor of the client, take the following arguments and store them in the client instance for use in later requests:

**Arguments:**

| authSigner | Private key used to sign RPC requests sent to Flashbots. |
| --- | --- |
| streamUrl | URL to connect to SSE event stream (e.g. https://mev-share.flashbots.net) |
| apiUrl | URL to connect to Flashbots bundle API (e.g. https://relay.flashbots.net) |

## payload signature

POST requests sent to the Bundle API (e.g. `relay.flashbots.net`) must include a signature by `authSigner` (from constructor params) of the keccak256 hash of the request body, set in the header `X-Flashbots-Signature`. This is used for reputation on Flashbots.

Reference implementations are provided:

- [matchmaker-ts client](https://github.com/flashbots/matchmaker-ts/blob/main/src/flashbots.ts#L24) (typescript, ethers v6)
- [ethers-provider-flashbots-bundle client](https://github.com/flashbots/ethers-provider-flashbots-bundle/blob/master/src/index.ts#L1083) (typescript, ethers v5)
- [flashbotsrpc client](https://github.com/metachris/flashbotsrpc/blob/master/flashbotsrpc.go#L165-L171) (go)
- [artemis matchmaker client](https://github.com/paradigmxyz/artemis/blob/main/crates/clients/matchmaker/src/flashbots_signer.rs#L75-L78) (rust)

### encoding/serialization

Please encode the JSON serialization discretely in its own layer, so that it can be replaced with another encoding at a later time.

## public methods

The following methods are called from an instance of the mev-share client, using the params stored by the constructor to send requests. The names are only descriptions — please use your best judgement to find the simplest idiomatic naming scheme for your library’s methods.

***Note:** all functions which return a value must use the language’s async primitives.*

### event handlers

Listen to the SSE event stream* and trigger some user-defined code when a new event is detected. 

The client should allow the user to specify different code to be triggered when A) a bundle is detected, and B) a transaction is detected. An event with a `null` value in the `txs` field is considered a “transaction event”. Events with a non-null value for `txs` are considered “bundle events”. 

> * listening to the event stream may not necessarily start when this function is called, depending on your implementation. This is only to note that we have to be listening to the stream to trigger the provided code.
> 

*Examples for callback-based languages (yours does not have to be callback-oriented, this is just an example to demonstrate the required functionality):*

If using a `.on` pattern:

```tsx
function on(eventType: "transaction" | "bundle", callback: (event) => void)
```

> Note: if using a string pattern for the event type argument, please codify the argument type in a separate construct, such as an enum or a type alias (again, the actual definition and name of this primitive depends on your language). An example in Typescript can be found [here](https://github.com/flashbots/matchmaker-ts/blob/main/src/api/interfaces.ts#L6-L11).
> 

If using a verbose pattern:

```tsx
function onTransaction(callback: (event) => void) {}
function onBundle(callback: (event) => void) {}
```

**Return Value**

The function which sets the callback does not need to return anything (excluding runtime errors if it’s common in your language to return them).

### sendBundle

Send a bundle by calling `mev_sendBundle` on the instance’s `apiUrl` following the [mev_sendBundle spec](https://github.com/flashbots/mev-share/blob/main/specs/bundles/v0.1.md).

**Arguments**

| bundleParams | language-specific construct (struct/interface/type/object/etc) to represent the params in the https://github.com/flashbots/mev-share/blob/main/specs/bundles/v0.1.md#json-rpc-request-scheme |
| --- | --- |

**Return Value**

Bundle hash response, deserialized ([TS example](https://github.com/flashbots/matchmaker-ts/blob/main/src/api/interfaces.ts#L148-L151)).

### simulateBundle

This function simulates a bundle by calling `mev_simBundle` on the instance’s `apiUrl`. The important thing to note for this function is that `mev_simBundle` is not designed to simulate bundles which include transactions specified by `{ hash: "0x..." }`. These bundles are considered “unmatched”, and will throw an error from the RPC endpoint. The reason for this is that sharing simulation feedback about a user’s transaction will necessarily reveal more information to searchers than the user intended with the hints they set; it breaks user privacy guarantees.

So we have to wait for the user’s transaction to land on-chain, where all the data that was hidden by MEV-Share is now public. To prevent users from having to do this themselves, this method should wait for transactions specified by `hash`  in the bundle body to land on-chain, after which point, we fetch and serialize the full transaction and replace the `hash` transaction in the bundle body with the serialized transaction: `{ tx: serializedTx, canRevert: false }`, before finally calling `mev_simBundle` with the edited bundle body and returning the result.

**Arguments**

| bundleParams | same as bundleParams in sendBundle |
| --- | --- |
| simOptions | optional simulation parameters, which change the block context data in the simulation (see code snippet below for an implementation of these options in Typescript). |

```tsx
/** Optional fields to override simulation state. */
export interface SimBundleOptions {
    /** Block used for simulation state. Defaults to latest block.
     *
     * Block header data will be derived from parent block by default.
     * Specify other params in this interface to override the default values.
     *
     * Can be a block number or block hash.
    */
    parentBlock?: number | string,

    // override the default values for the parentBlock header
    /** default = parentBlock.number + 1 */
    blockNumber?: number,
    /** default = parentBlock.coinbase */
    coinbase?: string,
    /** default = parentBlock.timestamp + 12 */
    timestamp?: number,
    /** default = parentBlock.gasLimit */
    gasLimit?: number,
    /** default = parentBlock.baseFeePerGas */
    baseFee?: bigint,
    /** default = 5 (defined in seconds) */
    timeout?: number,
}
```

**Return Value**

Simulation response, deserialized ([TS example](https://github.com/flashbots/matchmaker-ts/blob/main/src/api/interfaces.ts#L191-L200)).

### getEventHistoryInfo

Fetch event history parameters by making an HTTP GET request to the instance’s `streamUrl` at the `/api/v1/history/info` endpoint.

**Return Value**

Event history info response, deserialized ([TS example](https://github.com/flashbots/matchmaker-ts/blob/main/src/api/interfaces.ts#L14-L21)).

### getEventHistory

Fetch event history by making an HTTP GET request to the instance’s `streamUrl` at the `/api/v1/history` endpoint.

This endpoint accepts optional parameters as HTTP query params.

The function should implement the optional parameters as language-native constructs for the function arguments.

*Typescript struct implementation:*

```tsx
export type EventHistoryParams = {
    blockStart?: number,
    blockEnd?: number,
    timestampStart?: number,
    timestampEnd?: number,
    limit?: number,
    offset?: number,
}
```

*Example request:*

https://mev-share.flashbots.net/api/v1/history?limit=10

**Arguments**

| options | optional parameters to refine the query |
| --- | --- |

**Return Value**

Event history response, deserialized ([TS example](https://github.com/flashbots/matchmaker-ts/blob/main/src/api/interfaces.ts#L34-L48)).

### sendTransaction

This method sends a transaction via `eth_sendPrivateTransaction` to the instance’s `apiUrl`.

**Arguments**

| signedTx | raw serialized transaction |
| --- | --- |
| options | private transaction options (specified below) |

```tsx
export interface TransactionOptions {
    /** Hints define what data about a transaction is shared with searchers. */
    hints?: HintPreferences,
    /** Maximum block number for the transaction to be included in. */
    maxBlockNumber?: number,
    builders?: string[],
}
```

**Return Value**

Transaction hash.