# Authentication

* `engine` JSON-RPC interface
  * exposed -- by -- execution layer clients
  * consumed -- by -- consensus layer clients
  * requires
    * authentication

* authentication scheme
  * == [JWT](https://jwt.io/)
  * attacks / this authentication scheme tries to protect against
    - RPC port 
      - -- exposed towards the -- internet
        - -> attackers can exchange messages -- with -- execution layer engine API
      - -- exposed towards the -- browser
        - -> malicious webpages can submit messages -- to the -- execution layer engine API
  * NOT designed to
    - prevent attackers / has capability to read ('sniff') network traffic
      - -- from -- reading the traffic
      - -- from -- performing replay-attacks of EARLIER messages

* way to authenticate
  - | `HTTP` dialogue,
    - EACH `jsonrpc` request is INDIVIDUALLY authenticated -- by supplying -- `JWT` token | HTTP header
  - | WebSocket dialogue, 
    * ONLY authenticate INITIAL handshake,
      * AFTER message dialogue proceeds -- without -- using JWT
    * clarification
      * websocket handshake's steps
        * consensus layer client perform a websocket upgrade request
    * == regular http GET request /
      * WS-handshake's actual parameters are carried | http headers
    * `inproc`
      * raw ipc communication
      * ❌NO require authentication❌
      * assumption 
        * process / able to access `ipc` channels -- for the -- process
          * == local file access
          * TODO: is already sufficiently permissioned that further authentication requirements do not add security.

## JWT specifications

- execution layer client
  - authenticated Engine API's
    - default port == "8551"
    * `engine` namespace
  - **MUST**
    - expose the authenticated Engine API | port independent
      - -- from -- EXISTING JSON-RPC API 
    - support: `alg` `HMAC + SHA256` (`HS256`)
    - reject the `alg` `none`

* HMAC algorithm /
  * SEVERAL consensus layer clients can use the SAME key
  * can impersonate -- , from an authentication perspective -- EACH other
  * | deployment perspective,
    * EL (TODO:❓) does NOT need to be provisioned -- with -- individual keys / EACH consensus layer client

## Key distribution

* execution layer & consensus layer clients
  * **SHOULD** accept a `jwt-secret`
    * == configuration parameter
    * == file / contains the hex-encoded 256 bit secret key
      * allows
        * verifying/generating JWT tokens
    * if SUCH parameter is NOT given -> the client 
      * **SHOULD** generate a token / valid | execution's duration
      * **SHOULD** store the hex-encoded secret | filesystem's `jwt.hex` file 
    * uses
      * provision the counterpart client
    * if it's configured & file can NOT be read OR NOT contain a hex-encoded key of `256` bits -> client **SHOULD** treat this as an error ==
      * abort the startup, or 
      * show error & continue WITHOUT exposing the authenticated port

## JWT Claims

* JWT claims / used | this specification
  - `iat` (issued-at) claim
    * REQUIRED
    * execution layer client **SHOULD** ONLY accept `iat` timestamps / [CURRENT time +-60 seconds]
  - `id` claim
    * OPTIONAL
    * uses
      * consensus layer client **MAY** use it -- to communicate an -- individual consensus layer client's unique identifier 
  - `clv` claim
    * OPTIONAL
    * uses
      * consensus layer client **MAY** use it -- to communicate the -- consensus layer client type/version

* if the execution layer client sees claims / it does NOT recognize -> these **MUST** be ignored
