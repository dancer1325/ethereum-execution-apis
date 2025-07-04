# Engine API -- Paris

* == Engine API's structure + methods / specified | Prague

## Structures

### ExecutionPayloadV1

* structure 
  * uses
    * | beacon chain spec [`ExecutionPayload`](https://github.com/ethereum/consensus-specs/blob/dev/specs/bellatrix/beacon-chain.md#ExecutionPayload) structure 
  * 's fields encoded
    - `parentHash`: `DATA`
      - 32 Bytes
    - `feeRecipient`:  `DATA`
      - 20 Bytes
    - `stateRoot`: `DATA`
      - 32 Bytes
    - `receiptsRoot`: `DATA`
      - 32 Bytes
    - `logsBloom`: `DATA`
      - 256 Bytes
    - `prevRandao`: `DATA`
      - 32 Bytes
    - `blockNumber`: `QUANTITY`
      - 64 Bits
    - `gasLimit`: `QUANTITY`
      - 64 Bits
    - `gasUsed`: `QUANTITY`
      - 64 Bits
    - `timestamp`: `QUANTITY`
      - 64 Bits
    - `extraData`: `DATA`
      - [0, 32] Bytes
    - `baseFeePerGas`: `QUANTITY`
      - 256 Bits
    - `blockHash`: `DATA`
      - 32 Bytes
    - `transactions`: `Array of DATA`
      - == [transaction objects]
        - EACH object == byte list (`DATA`) /
          - ALLOWED types: `TransactionType || TransactionPayload` or `LegacyTransaction` 
            - see [EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)

### ForkchoiceStateV1

* encapsulates the fork choice state
* 's fields encoded
  - `headBlockHash`: `DATA`
    - 32 Bytes
    - canonical chain's head's block hash 
  - `safeBlockHash`: `DATA`
    - 32 Bytes
    - canonical chain's "safe" block hash / has
      - synchrony assumptions
      - honesty assumptions
    - **MUST** be 
      - == `headBlockHash` OR
      - ancestor of `headBlockHash`  
    - if transition block is NOT finalized -> can have `0x0000000000000000000000000000000000000000000000000000000000000000` value
  - `finalizedBlockHash`: `DATA`
    - 32 Bytes
    - MOST recent finalized block's block hash
    - if transition block is NOT finalized -> can have `0x0000000000000000000000000000000000000000000000000000000000000000` value

### PayloadAttributesV1

* 's attributes
  * allows
    * initiate a payload build process | `engine_forkchoiceUpdated` call
* 's fields encoded
  - `timestamp`: `QUANTITY`
    - 64 Bits
    - NEW payload's `timestamp` field 
  - `prevRandao`: `DATA`
    - 32 Bytes
    - NEW payload's `prevRandao` field
  - `suggestedFeeRecipient`: `DATA`
    - 20 Bytes
    - NEW payload's suggested `feeRecipient` field

### PayloadStatusV1

* == result of processing a payload
* 's fields encoded
  - `status`: `enum`
    - == `"VALID" | "INVALID" | "SYNCING" | "ACCEPTED" | "INVALID_BLOCK_HASH"`
  - `latestValidHash`: `DATA|null`
    - 32 Bytes
    - MOST RECENT *valid* block's hash | branch / defined -- by -- payload + its ancestors
  - `validationError`: `String|null`
    - if the payload == `INVALID` or `INVALID_BLOCK_HASH` -> message / provide ADDITIONAL details ABOUT validation error 

### TransitionConfigurationV1

* transition process' configurable settings 
* 's fields encoded
  - `terminalTotalDifficulty`: `QUANTITY`
    - 256 Bits
    - [EIP-3675](https://eips.ethereum.org/EIPS/eip-3675#client-software-configuration)'s parameter `TERMINAL_TOTAL_DIFFICULTY`   
  - `terminalBlockHash`: `DATA`
    - 32 Bytes
    - [EIP-3675](https://eips.ethereum.org/EIPS/eip-3675#client-software-configuration)'s parameter `TERMINAL_BLOCK_HASH` 
  - `terminalBlockNumber`: `QUANTITY`
    - 64 Bits
    - [EIP-3675](https://eips.ethereum.org/EIPS/eip-3675#client-software-configuration)'s parameter `TERMINAL_BLOCK_NUMBER`

## Routines

### Payload validation

* == validate a payload -- with respect to the -- block header + execution environment rule sets

1. Client software **MAY** obtain -- , by executing payload's ancestors -- as a -- part of the validation process, -- parent state 
  * EACH ancestor **MUST** ALSO pass payload validation process
2. Client software **MUST** validate: MOST recent PoW block | chain of a payload ancestors satisfies terminal block conditions / according to [EIP-3675](https://eips.ethereum.org/EIPS/eip-3675#transition-block-validity)
  * if this validation fails -> the response **MUST** contain `{status: INVALID, latestValidHash: 0x0000000000000000000000000000000000000000000000000000000000000000}`
  * EACH block | tree of descendants of an invalid terminal block **MUST** be deemed `INVALID`
3. Client software **MUST** validate a payload -- according to the -- block header & execution environment rule set / 
  * modifications to these rule sets
    * defined | [EIP-3675's Block Validity](https://eips.ethereum.org/EIPS/eip-3675#block-validity)
    * if validation succeeds -> response **MUST** contain `{status: VALID, latestValidHash: payload.blockHash}`
    * if validation fails -> response **MUST** contain `{status: INVALID, latestValidHash: validHash}` / 
      - `validHash` **MUST** be == invalid payload's ancestor's block hash / 
        - satisfy
          - FULLY validated & deemed `VALID`
          - ANY invalid payload's OTHER ancestor / > `blockNumber` is `INVALID`
        - if ABOVE conditions are satisfied / PoW block -> `0x0000000000000000000000000000000000000000000000000000000000000000` 
      - if client software can NOT determine the invalid payload's ancestor / satisfy these conditions -> `null`
  * Client software **MUST NOT** surface an `INVALID` payload -- over -- ANY API endpoint & p2p interface
4. Payload validation process **MUST** be idempotent -- respect to -- payload's validity status (`VALID | INVALID`)
   * == | validation process time, payload / validity status `INVALID (INVALID_BLOCK_HASH)` **MUST NOT** <- become -> `VALID` 
   * if | validation process time, client software payload REMAINS `INVALID` -> **MAY** change payload status FROM `INVALID` -- to -- `SYNCING | ACCEPTED`
5. if a payload is deemed `INVALID` | assign the corresponding message -- to the -- `validationError` field -> client software **MAY** provide additional details | validation error 
6. validate a payload | canonical chain, **MUST NOT** be affected -- by an -- active sync process | block tree's side branch 
   * _Example:_ if side branch `B` is `SYNCING` BUT requisite data / validatE a payload FROM canonical branch `A` is available -> client software **MUST** 
     * run FULL validation of the payload
     * respond accordingly

### Sync

* sync
  * == process of obtaining data / validate a payload
    * 👀EXACT behavior depends -- on -- EACH implementation👀
  * POSSIBLE stages
    1. Pulling data FROM remote peers | network
       * OPTIONAL
    2. Passing ancestors of a payload -- through the -- [Payload validation](#payload-validation) & obtain a parent state
       * OPTIONAL

### Payload building

1. Client software **MUST** set the payload field values -- according to the -- set of parameters / passed | this method
   * EXCEPT TO, `suggestedFeeRecipient`
   * `ExecutionPayload` **MAY** deviate the `feeRecipient` field value -- from -- specified | `suggestedFeeRecipient` parameter

2. Client software **SHOULD** build the payload's initial version / has an EMPTY transaction set

3. Client software **SHOULD** start the process of updating the payload
   * strategy
     * implementation dependent
   * default strategy
     * keep the transaction set up-to-date -- with the -- local mempool's state

4. scenario / Client software **SHOULD** stop the updating process
   * call to `engine_getPayload` / build process's `payloadId` is made OR
   * `timestamp` parameter + [`SECONDS_PER_SLOT`](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#time-parameters-1) have ALREADY passed 
     * (| Mainnet configuration, 12s) 

5. if `PayloadAttributes` does NOT match EXISTING build process' payload attributes -> Client software **MUST** begin a NEW build process 
   * ALL NEW build process **MUST** be uniquely identified -- by the -- returned `payloadId`

6. if a build process / `PayloadAttributes` ALREADY exists -> client software **SHOULD NOT** restart it

## Methods

### `engine_newPayloadV1`

#### Request

* params:
  1. [`ExecutionPayloadV1`](#ExecutionPayloadV1)
* timeout: 8s

#### Response

* result: [`PayloadStatusV1`](#PayloadStatusV1)
* error: code + message
  * | process the payload, exception happens 

#### Specification

1. Client software **MUST** validate: ALL `transactions` have  length !=0 (>= 1 byte)
   * run | ALL cases
     * EVEN if this branch OR ANY block tree's branches are | active sync process

2. Client software **MUST** validate: `blockHash` value == `Keccak256(RLP(ExecutionBlockHeader))`
   * `ExecutionBlockHeader` == execution layer block header (== FORMER PoW block header structure)
     * 's fields == corresponding payload values + constant values /
       * -- according to the -- [EIP-3675's Block structure section](https://eips.ethereum.org/EIPS/eip-3675#block-structure) & [EIP-4399](https://eips.ethereum.org/EIPS/eip-4399#block-structure)
   * run | ALL cases
     * EVEN if this branch OR ANY block tree's branches are | active sync process

3. if requisite data for payload validation is missing -> Client software **MAY** initiate a sync process

4. if payload extends the canonical chain & requisite data for the validation is LOCALLY AVAILABLE -> Client software **MUST** validate the payload 
   * validation process is specified | [Payload validation](#payload-validation)

5. if the payload does NOT belong to the canonical chain -> Client software **MAY NOT** validate the payload 

6. Client software **MUST** respond to this method call
  * if `transactions` contains 0 length OR invalid entries -> `{status: INVALID, latestValidHash: null, validationError: errorMessage | null}` 
  * if the `blockHash` validation has failed -> `{status: INVALID_BLOCK_HASH, latestValidHash: null, validationError: errorMessage | null}` 
  * if terminal block conditions are NOT satisfied -> `{status: INVALID, latestValidHash: 0x0000000000000000000000000000000000000000000000000000000000000000, validationError: errorMessage | null}` 
  * if requisite data for the payload's acceptance or validation is missing -> `{status: SYNCING, latestValidHash: null, validationError: null}` 
  * if the payload has been FULLY validated | process the call -> payload status obtained -- from the -- [Payload validation](#payload-validation) process 
  * `{status: ACCEPTED, latestValidHash: null, validationError: null}`, if following conditions are met
    - ALL `transactions`'s length !=0
    - payload
      - 's `blockHash` is valid
      - does NOT extend the canonical chain
      - has NOT been fully validated
      - 's ancestors 
        - are known
        - == well-formed chain

7. if ANY of the ABOVE fails -- due to -- errors / unrelated to the normal processing flow -> Client software **MUST** respond -- with an -- error object

### engine_forkchoiceUpdatedV1

#### Request

* params:
  1. `forkchoiceState`: `Object`
     * == [`ForkchoiceStateV1`](#ForkchoiceStateV1) instance
  2. `payloadAttributes`: `Object|null`
     * == [`PayloadAttributesV1`](#PayloadAttributesV1) instance OR `null`
* timeout: 8s

#### Response

* result: `object`
  - `payloadStatus`: [`PayloadStatusV1`](#PayloadStatusV1)
    - ALLOWED values
      * `"VALID"`
      * `"INVALID"`
      * `"SYNCING"`
  - `payloadId`: `DATA|null`
    - 8 Bytes
    - payload build process identifier

* error: code + message
  * exception happens, |
    * validate payload,
    * update the forkchoice
    * initiate the payload build process

#### Specification

1. if `forkchoiceState.headBlockHash` -- references -- an unknown payload OR payload / can NOT be validated (Reason: 🧠data / required for the validation is missing🧠) -> Client software **MAY** initiate a sync process

2. TODO: Client software **MAY** skip an update of the forkchoice state and **MUST NOT** begin a payload build process if `forkchoiceState.headBlockHash` references a `VALID` ancestor of the head of canonical chain, i.e. the ancestor passed [payload validation](#payload-validation) process and deemed `VALID`
* In the case of such an event, client software **MUST** return `{payloadStatus: {status: VALID, latestValidHash: forkchoiceState.headBlockHash, validationError: null}, payloadId: null}`.

3. If `forkchoiceState.headBlockHash` references a PoW block, client software **MUST** validate this block with respect to terminal block conditions according to [EIP-3675](https://eips.ethereum.org/EIPS/eip-3675#transition-block-validity)
* This check maps to the transition block validity section of the EIP
* Additionally, if this validation fails, client software **MUST NOT** update the forkchoice state and **MUST NOT** begin a payload build process.

4. Before updating the forkchoice state, client software **MUST** ensure the validity of the payload referenced by `forkchoiceState.headBlockHash`, and **MAY** validate the payload while processing the call
* The validation process is specified in the [Payload validation](#payload-validation) section
* If the validation process fails, client software **MUST NOT** update the forkchoice state and **MUST NOT** begin a payload build process.

5. Client software **MUST** update its forkchoice state if payloads referenced by `forkchoiceState.headBlockHash` and `forkchoiceState.finalizedBlockHash` are `VALID`
* The update is specified as follows:
  * The values `(forkchoiceState.headBlockHash, forkchoiceState.finalizedBlockHash)` of this method call map on the `POS_FORKCHOICE_UPDATED` event of [EIP-3675](https://eips.ethereum.org/EIPS/eip-3675#block-validity) and **MUST** be processed according to the specification defined in the EIP
  * All updates to the forkchoice state resulting from this call **MUST** be made atomically.

6. Client software **MUST** return `-38002: Invalid forkchoice state` error if the payload referenced by `forkchoiceState.headBlockHash` is `VALID` and a payload referenced by either `forkchoiceState.finalizedBlockHash` or `forkchoiceState.safeBlockHash` does not belong to the chain defined by `forkchoiceState.headBlockHash`.

7. Client software **MUST** process provided `payloadAttributes` after successfully applying the `forkchoiceState` and only if the payload referenced by `forkchoiceState.headBlockHash` is `VALID`
* The processing flow is as follows:

    1. Verify that `payloadAttributes.timestamp` is greater than `timestamp` of a block referenced by `forkchoiceState.headBlockHash` and return `-38003: Invalid payload attributes` on failure.

    2. If `payloadAttributes` passes all validation steps, begin a payload build process building on top of `forkchoiceState.headBlockHash` and identified via `buildProcessId` value. The build process is specified in the [Payload building](#payload-building) section.

    3. If `payloadAttributes` validation fails, the `forkchoiceState` update **MUST NOT** be rolled back.

8. Client software **MUST** respond to this method call in the following way:
  * `{payloadStatus: {status: SYNCING, latestValidHash: null, validationError: null}, payloadId: null}` if `forkchoiceState.headBlockHash` references an unknown payload or a payload that can't be validated because requisite data for the validation is missing
  * `{payloadStatus: {status: INVALID, latestValidHash: validHash, validationError: errorMessage | null}, payloadId: null}` obtained from the [Payload validation](#payload-validation) process if the payload is deemed `INVALID`
  * `{payloadStatus: {status: INVALID, latestValidHash: 0x0000000000000000000000000000000000000000000000000000000000000000, validationError: errorMessage | null}, payloadId: null}` obtained either from the [Payload validation](#payload-validation) process or as a result of validating a terminal PoW block referenced by `forkchoiceState.headBlockHash`
  * `{payloadStatus: {status: VALID, latestValidHash: forkchoiceState.headBlockHash, validationError: null}, payloadId: null}` if the payload is deemed `VALID` and a build process hasn't been started
  * `{payloadStatus: {status: VALID, latestValidHash: forkchoiceState.headBlockHash, validationError: null}, payloadId: buildProcessId}` if the payload is deemed `VALID` and the build process has begun
  * `{error: {code: -38002, message: "Invalid forkchoice state"}}` if `forkchoiceState` is either invalid or inconsistent
  * `{error: {code: -38003, message: "Invalid payload attributes"}}` if the payload is deemed `VALID` and `forkchoiceState` has been applied successfully, but no build process has been started due to invalid `payloadAttributes`.

9. If any of the above fails due to errors unrelated to the normal processing flow of the method, client software **MUST** respond with an error object.

### engine_getPayloadV1

#### Request

* method: `engine_getPayloadV1`
* params:
  1. `payloadId`: `DATA`, 8 Bytes - Identifier of the payload build process
* timeout: 1s

#### Response

* result: [`ExecutionPayloadV1`](#ExecutionPayloadV1)
* error: code and message set in case an exception happens while getting the payload.

#### Specification

1. Given the `payloadId` client software **MUST** return the most recent version of the payload that is available in the corresponding build process at the time of receiving the call.

2. The call **MUST** return `-38001: Unknown payload` error if the build process identified by the `payloadId` does not exist.

3. Client software **MAY** stop the corresponding build process after serving this call.

### engine_exchangeTransitionConfigurationV1

#### Request

* method: `engine_exchangeTransitionConfigurationV1`
* params:
  1. `transitionConfiguration`: `Object` - instance of [`TransitionConfigurationV1`](#TransitionConfigurationV1)
* timeout: 1s

#### Response

* result: [`TransitionConfigurationV1`](#TransitionConfigurationV1)
* error: code and message set in case an exception happens while getting a transition configuration.

#### Specification

1. Execution Layer client software **MUST** respond with configurable setting values that are set according to the Client software configuration section of [EIP-3675](https://eips.ethereum.org/EIPS/eip-3675#client-software-configuration).

2. Execution Layer client software **SHOULD** surface an error to the user if local configuration settings mismatch corresponding values received in the call of this method, with exception for `terminalBlockNumber` value.

3. Consensus Layer client software **SHOULD** surface an error to the user if local configuration settings mismatch corresponding values obtained from the response to the call of this method.

4. Consensus Layer client software **SHOULD** poll this endpoint every 60 seconds.

5. Execution Layer client software **SHOULD** surface an error to the user if it does not receive a request on this endpoint at least once every 120 seconds.

6. Considering the absence of the `TERMINAL_BLOCK_NUMBER` setting, Consensus Layer client software **MAY** use `0` value for the `terminalBlockNumber` field in the input parameters of this call.

7. Considering the absence of the `TERMINAL_TOTAL_DIFFICULTY` value (i.e. when a value has not been decided), Consensus Layer and Execution Layer client software **MUST** use `115792089237316195423570985008687907853269984665640564039457584007913129638912` value (equal to`2**256-2**10`) for the `terminalTotalDifficulty` input parameter of this call.
