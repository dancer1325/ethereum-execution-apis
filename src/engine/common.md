# Engine API -- Common Definitions & requirements

## Underlying protocol

* message format & encoding notation are inherited -- from -- [Ethereum JSON-RPC Specification](https://playground.open-rpc.org/?schemaUrl=https://raw.githubusercontent.com/ethereum/execution-apis/assembled-spec/openrpc.json&uiSchema[appBar][ui:splitView]=false&uiSchema[appBar][ui:input]=false&uiSchema[appBar][ui:examplesDropdown]=false)

* Client software 
  * **MUST** expose 
    * Engine API | port 
      * -- independent from -- JSON-RPC API
      * default port: "8551"
      * | "engine" namespace
    * `eth` methods
      * `eth_blockNumber`
      * `eth_call`
      * `eth_chainId`
      * `eth_getCode`
      * `eth_getBlockByHash`
      * `eth_getBlockByNumber`
      * `eth_getLogs`
      * `eth_sendRawTransaction`
      * `eth_syncing`
      * Reason:🧠facilitate an Engine API consumer can access -- , through the SAME connection, -- state & logs (_Example:_ proof-of-stake deposits)🧠
      * see [Ethereum JSON-RPC Specification](https://playground.open-rpc.org/?schemaUrl=https://raw.githubusercontent.com/ethereum/execution-apis/assembled-spec/openrpc.json&uiSchema[appBar][ui:splitView]=false&uiSchema[appBar][ui:input]=false&uiSchema[appBar][ui:examplesDropdown]=false)

### Authentication

* [Authentication](./authentication.md)

## Versioning

* method & structure are 👀versioned INDEPENDENTLY👀
  * `VX` / `X` == version number
    * 👀's suffix -- related to -- EACH method & structure's name👀
      * specification **MAY** reference a method or a structure WITHOUT it
        * _Example:_ `engine_newPayload` 
  * +1, if it's changed
    * method parameters
    * method response value
    * method behavior
    * structure fields

## Message ordering

* Consensus Layer client software
  * **MUST** -- respect -- corresponding fork choice update events | make calls to the `engine_forkchoiceUpdated`

* Execution Layer client software
  * **MUST** process `engine_forkchoiceUpdated` method calls' order == order / they have been received

## Load-balancing and advanced configurations

* Engine API 
  * supports
    * 1-to-many: Consensus Layer -- to -- Execution Layer configuration
      * Reason:🧠Consensus Layer drives the Execution Layer -> Consensus Layer can drive many of them INDEPENDENTLY🧠
  * ❌NOT support❌
    * generic many-to-1 Consensus Layer -- to -- Execution Layer configurations
      * Reason: 🧠Execution Layer, by default, ONLY supports 1 chain head | time🧠
      * if ONLY 1 Consensus Layer instantiation is able to *write* | Execution Layer's chain head & initiate the payload build process (== call `engine_forkchoiceUpdated` ) -> Engine API works properly
        * == OTHER Consensus Layers ONLY insert payloads (== `engine_newPayload`) & read -- from the -- Execution Layer

## Errors

* error codes / introduced -- by -- this specification

| Code   | Message                    | Meaning                                          |
|--------|----------------------------|--------------------------------------------------|
| -32700 | Parse error                | Invalid JSON was received by the server.         |
| -32600 | Invalid Request            | The JSON sent is not a valid Request object.     |
| -32601 | Method not found           | The method does not exist / is not available.    |
| -32602 | Invalid params             | Invalid method parameter(s).                     |
| -32603 | Internal error             | Internal JSON-RPC error.                         |
| -32000 | Server error               | Generic client error while processing request.   |
| -38001 | Unknown payload            | Payload does not exist / is not available.       |
| -38002 | Invalid forkchoice state   | Forkchoice state is invalid / inconsistent.      |
| -38003 | Invalid payload attributes | Payload attributes are invalid / inconsistent.   |
| -38004 | Too large request          | Number of requested entities is too large.       |
| -38005 | Unsupported fork           | Payload belongs to a fork that is not supported. |

* ALL errors, EXCEPT TO `-32000`
  * returns `null` `data` value
* `-32000`
  * returns the `data` object / `err` member / explains the error encountered
  * _Example:_

    ```console
    $ curl https://localhost:8551 \
        -X POST \
        -H "Content-Type: application/json" \
        -d '{"jsonrpc":"2.0","method":"engine_getPayloadV1","params": ["0x1"],"id":1}'
    {
      "jsonrpc": "2.0",
      "id": 1,
      "error": {
        "code": -32000,
        "message": "Server error",
        "data": {
            "err": "Database corrupted"
        }
      }
    }
    ```

## Timeouts

* TODO: Consensus Layer client software **MUST** wait for a specified `timeout` before aborting the call. In such an event, the Consensus Layer client software **SHOULD** retry the call when it is needed to keep progressing.

Consensus Layer client software **MAY** wait for response longer than it is specified by the `timeout` parameter.

## Encoding

Values of a field of `DATA` type **MUST** be encoded as a hexadecimal string with a `0x` prefix matching the regular expression `^0x(?:[a-fA-F0-9]{2})*$`.

Values of a field of `QUANTITY` type **MUST** be encoded as a hexadecimal string with a `0x` prefix and the leading 0s stripped (except for the case of encoding the value `0`) matching the regular expression `^0x(?:0|(?:[a-fA-F1-9][a-fA-F0-9]*))$`.

*Note:* Byte order of encoded value having `QUANTITY` type is big-endian.

[json-rpc-spec]: 

## Capabilities

Execution and consensus layer client software may exchange with a list of supported Engine API methods by calling `engine_exchangeCapabilities` method.

Execution layer clients **MUST** support `engine_exchangeCapabilities` method, while consensus layer clients are free to choose whether to call it or not.

*Note:* The method itself doesn't have a version suffix.

### engine_exchangeCapabilities

#### Request

* method: `engine_exchangeCapabilities`
* params:
    1. `Array of string` -- Array of strings, each string is a name of a method supported by consensus layer client software.
* timeout: 1s

#### Response

`Array of string` -- Array of strings, each string is a name of a method supported by execution layer client software.

#### Specification

1. Consensus and execution layer client software **MAY** exchange with a list of currently supported Engine API methods. Execution layer client software **MUST NOT** log any error messages if this method has either never been called or hasn't been called for a significant amount of time.

2. Request and response lists **MUST** contain Engine API methods that are currently supported by consensus and execution client software respectively. Name of each method in both lists **MUST** include suffixed version. Consider the following examples:
    * Client software of both layers currently supports `V1` and `V2` versions of `engine_newPayload` method:
        * params: `[["engine_newPayloadV1", "engine_newPayloadV2", ...]]`,
        * response: `["engine_newPayloadV1", "engine_newPayloadV2", ...]`.
    * `V1` method has been deprecated and `V3` method has been introduced on execution layer side since the last call:
        * params: `[["engine_newPayloadV1", "engine_newPayloadV2", ...]]`,
        * response: `["engine_newPayloadV2", "engine_newPayloadV3", ...]`.
    * The same capabilities modification has happened in consensus layer client, so, both clients have the same capability set again:
        * params: `[["engine_newPayloadV2", "engine_newPayloadV3", ...]]`,
        * response: `["engine_newPayloadV2", "engine_newPayloadV3", ...]`.

3. The `engine_exchangeCapabilities` method **MUST NOT** be returned in the response list.
