# Engine API -- Prague

* == Engine API changes / introduced | Prague
  * 💡based on & extends [Engine API - Cancun specification](./cancun.md)💡

## Methods

### engine_newPayloadV4

* it's extended -- with -- `executionRequests`

#### Request

* params:
  1. `executionPayload`: [`ExecutionPayloadV3`](./cancun.md#executionpayloadv3)
  2. `expectedBlobVersionedHashes`: `Array of DATA`
     1. 32 Bytes
     2. [] expected blob versioned hashes -- to -- validate
  3. `parentBeaconBlockRoot`: `DATA`
     1. 32 Bytes
     2. root of the parent beacon block
  4. `executionRequests`: `Array of DATA` 
     * list of execution layer triggered requests
     * EACH `requests` byte array's element -- is defined by -- [EIP-7685](https://eips.ethereum.org/EIPS/eip-7685)
       * `request_type`
         * == EACH element's FIRST byte 
       * `request_data`
         * == EACH element's REMAINING bytes 
       * elements **MUST** be ordered -- by -- `request_type` in ascending order
       * elements / empty `request_data` -> **MUST** be excluded from the list
       * if the list has NO elements -> expected array MUST be `[]`
       * if any element is out of order (== length of 1-byte or shorter) OR >1 element has the SAME type byte OR the param is `null` -> client software **MUST** return `-32602: Invalid params` error

#### Response

* == [`engine_newPayloadV3`'s response](./cancun.md#engine_newpayloadv3)

#### Specification

* == [`engine_newPayloadV3`'s specification](./cancun.md#engine_newpayloadv3) /
  1. if the payload's `timestamp` does NOT fall | Prague fork's time frame -> **MUST** return `-38005: Unsupported fork` error 
  2. FROM `executionRequests` -> client software **MUST**
     * compute the execution requests commitment
     * incorporate the execution requests commitment | `blockHash` validation process
        * ❌if the computed commitment does NOT match the corresponding commitment | execution layer block header -> call **MUST** return `{status: INVALID, latestValidHash: null, validationError: errorMessage | null}`❌

### engine_getPayloadV4

* 's response
  * \+ `executionRequests` field

#### Request

* params:
  1. `payloadId`: `DATA`
     * 8 Bytes
     * payload build process's identifier 
* timeout: 1s

#### Response

* result: `object`
  - `executionPayload`: [`ExecutionPayloadV3`](./cancun.md#executionpayloadv3)
  - `blockValue` : `QUANTITY`
    - 256 Bits
    - == expected value / be received -- by the -- `feeRecipient`
      - [wei]
  - `blobsBundle`: [`BlobsBundleV1`](#BlobsBundleV1)
    - == bundle / has data -- corresponding to -- blob transactions included | `executionPayload`
  - `shouldOverrideBuilder` : `BOOLEAN`
    - suggested -- from the -- execution layer
    - if `true` == use this `executionPayload`
      - instead of EXTERNAL provided one
  - `executionRequests`: `Array of DATA`
    - triggered -- by -- execution layer
    - obtained -- from the -- `executionPayload` transaction execution
    - EACH `requests` byte array's element -- is defined by -- [EIP-7685](https://eips.ethereum.org/EIPS/eip-7685)
* error
  * code + message
  * requirements
    * | get the payload, exception happens 

#### Specification

* == [`engine_getPayloadV3`'s specification](./cancun.md#engine_getpayloadv3) + changes
  1. if the built payload `timestamp` does NOT fall | Prague fork's time frame -> client software **MUST** return `-38005: Unsupported fork` error 
  2. **MUST** return `executionRequests`

### Update the methods of previous forks

* goal
  * how Prague payload should be handled -- by the -- [`Cancun API`](./cancun.md)

* | [`engine_newPayloadV3`](./cancun.md#engine_newpayloadV3) & [`engine_getPayloadV3`](./cancun.md#engine_getpayloadv3),
  * a validation **MUST** be added
    1. if the payload's `timestamp` >= Prague activation timestamp -> Client software **MUST** return `-38005: Unsupported fork` error 

* | [`engine_forkchoiceUpdatedV3`](./cancun.md#engine_forkchoiceupdatedv3),
  * modification **MUST** be made 
    1. if `payloadAttributes.timestamp` does NOT fall | Cancun *or Prague* forks' time frame -> return `-38005: Unsupported fork` 
