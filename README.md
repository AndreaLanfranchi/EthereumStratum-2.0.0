# EthereumStratum/2.0.0 - Implementation proposal guide lines

## Abstract
This draft contains the guidelines to define a new standard for the Stratum protocol used by Ethereum miners to communicate with mining pool servers.

### Conventions
The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHALL NOT`, `SHOULD`, `SHOULD NOT`, `RECOMMENDED`, `MAY`, and `OPTIONAL` in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).
The definition `mining pool server`, and it's plural form, is to be interpreted as `work provider` and later in this document can be shortened as `pool` or `server`.
The definition `miner(s)`, and it's plural form, is to be interpreted as `work receiver/processor` and later in this document can be shortened as `miner` or `client`.

### Rationale
Ethereum does not have an official Stratum implementation yet. It officially supports only getWork which requires miners to constantly pool the work provider. Only recently go-ethereum has implemented a [push mechanism](https://github.com/ethereum/go-ethereum/pull/17347) to notify clients for mining work, but whereas the vast majority of miners do not run a node, its main purpose is to facilitate mining pools rather than miners.

The Stratum protocol on the other hand relies on a standard stateful TCP connection which allows two-way exchange of line-based messages. Each line contains the string representation of a JSON object following the rules of either [JSON-RPC 1.0](https://www.jsonrpc.org/specification_v1) or [JSON-RPC 2.0](https://www.jsonrpc.org/specification).

In absence of a well defined standard, various flavours of Stratum have bloomed for Ethereum mining as a derivative work for different mining pools implementations. The only attempt to define a standard was made by NiceHash with their [EthereumStratum/1.0.0](https://github.com/nicehash/Specifications/blob/master/EthereumStratum_NiceHash_v1.0.0.txt) implementation which is the main source this work inspires from.

Mining activity, thus the interaction among pools and miners, is at its basics very simple, and can be summarized with "_please find a number (nonce) which coupled to this data as input for a given hashing algorithm produces, as output, a result which is below a certain target_". Other messages may be exchanged among parties during a session, in order to support this concept.

In order to keep things simple and not introduce new dependencies, we will stick with JSON-RPC formatted objects.

### Stratum design flaws
Lacking a well defined standard, mining software developers have to struggle with different flavours of Stratum. When switching from one pool to another or even when trying to "guess" which is the flavour implemented by a single pool, developers have to implement workaround for variations in implementations.

All existing implementations suffer from excessive verbosity for a chain with a very small block time like Ethereum. A normal `mining.notify` message weighs roughly 240 bytes: assuming the dispatch of 1 work per block to an audience of 50k connected TCP sockets means the transmission of roughly 1.88TB of data a month. This can be an issue for large pools. From a miner's perspective, we understand how mining decentralization is heavily affected by the quality of internet connections.

### Sources of inspiration
- [NiceHash EthereumStratum/1.0.0](https://github.com/nicehash/Specifications/blob/master/EthereumStratum_NiceHash_v1.0.0.txt)
- [Zcash variant of the Stratum protocol](https://github.com/zcash/zips/blob/23d74b0373c824dd51c7854c0e3ea22489ba1b76/drafts/str4d-stratum/draft1.rst#json-rpc-1-0)
- [BetterHash Mining Protocol](https://github.com/TheBlueMatt/bips/blob/master/bip-XXXX.mediawiki)

## Specification
The Stratum protocol is an instance of [JSON-RPC-2.0](https://www.jsonrpc.org/specification). The miner is a JSON-RPC client, and the server is a JSON-RPC server. All communications exist within the scope of a `session`. A session starts the moment a client opens a TCP connection to the server, until the moment either party voluntary close the very same connection or it gets broken. Servers **MAY** support session resuming if this is initially negotiated (on first session handshaking) between the client and the server. During a session all messages exchanged among server and client are line-based. This means all messages are JSON strings terminated by ASCII LF character (which may also be denoted as `\n` in this document). The LF character **MUST NOT** appear elsewhere in a message. Client and server implementations **MUST** assume that once they read a LF character, the current message has been completely received and can be processed.

Line messages are of three types:

- `Requests` : messages for which the issuer expects a response. The receiver **MUST** reply back to any request individually
- `Responses` : solicited messages by a previous request. The responder **MUST** label the response with the same identifier of the originating request.
- `Notifications` : unsolicited messages for which the issuer is not interested nor is expecting a response. Nevertheless a response (eg. an aknowledgement) **MAY** be sent by the receiver.

During a `session` both parties **CAN** exchange messages of the above depicted three types.

### JSON-RPC-2.0 Compliances

As per [JSON-RPC-2.0](https://www.jsonrpc.org/specification) specification _requests_ and _responses_ differ from _notifications_ by the identifier (`id`) member in the JSON object: 
- Requests **MUST** have an `id` member
- Responses **MUST** have an `id` member valued exactly as the `id` member of the request this response is for
- Notifications **MUST NOT** have an `id` member

### JSON-RPC-2.0 Differences

In order to get the most concise messages among parties of a session/conversation this implementation enforces the following definitions:
- JSON member `jsonrpc` (always valued to "2.0") **MUST ALWAYS BE OMITTED**
- JSON member `id` **MUST NOT** be `null` and is mandatory in requests and responses
- JSON member `id` **MUST** be valued to a range from 0 to 65535. Please note that a message with `"id": 0` **MUST NOT** be interpreted as a notification: it's a request with identifier 0. The removal of other identifier types (namely strings) is due to the need to reduce the number of bytes transferred.

## Conventions
- The representation of a JSON object is, at its base, a string 
- The conversation among the client and the server is made of strings. Each string ending with a LF (ASCII char 10) denotes a `line`. Each line **MUST** contain only one JSON root object. Eventually the root object **MAY** contain additional JSON objects in its members.
- Aside the `LF` delimiter each `line` **MUST** be made of printable ASCII chars in the range 32..126
- It's implicit and mandatory that each line message corresponds to a well formatted JSON object: see [JSON documentation](https://www.json.org/)
- JSON objects are made of `members` which can be of type : primitive of string/number, JSON object, JSON arrays
- JSON `member`'s names are strings which **MUST** be composed of printable chars only in the ASCII range 48..57 (numbers) and 97..122 (lower case letters).
- JSON `member`'s names **MUST NOT** begin with a number.
- JSON values `arrays` : although the JSON notation allows the insertion of different data types within the same array, this behavior is generally not accepted in coding languages. Implementers **MUST** assume that an array is made of elements of the very same data type.
- JSON values `booleans` : the JSON notation allows to express boolean values as `true` or `false`. In EthereumStratum/2.0.0, for better compatibility within arrays, boolean values will be expressed in the hex form of "0" (false) or "1" (true)
- JSON values `strings` : any string value **MUST** be composed of printable ASCII chars only in the ASCII range 32..126. Each string is delimited by a `"` (ASCII 34) at the beginning and at the end. Should the string value contain a `"` this must be escaped as `\"`
- Hex values : a Hexadecimal representation of a number is actually a string data type. As convention and to reduce the number of transmitted bytes, the prefix "0x" **MUST** always be omitted. In addition, any hex number **MUST** be transferred only for their significant part i.e. the non significant zeroes **MUST** be omitted (example : the decimal `456` must be represented as `"1c8"` and not as `"01c8"` although the conversion produces the same decimal value). This directive **DOES NOT APPLY** to hashes and extranonces
- Hex values casing : all letters in Hexadecimal values **MUST** be lower case. (example : the decimal `456` must be represented as `"1c8"` and not as `"1C8"` although the conversion produces the same decimal value). This directive **DOES NOT APPLY** to hashes.
- Numbers : any non-fractional number **MUST** be transferred by its hexadecimal representation

### Requests
The JSON representation of `request` object is made of these parts:
- mandatory `id` member : the identifier established by the issuer. valued to a range from 0 to 65535.
- mandatory `method` member of type String : the name of the method to be invoked on the receiver side
- optional `params` member : in case the method invocation on the receiver side requires the application of additional parameters to be executed. The type **CAN** be Object (with named members of different types) or Array of single type. In case of an array, the parameters will be applied by their ordinal position. If the method requested for invocation on the receiver side does not require the application of additional parameters this member **MUST NOT** be present. The notation `"params" : null` **IS NOT PERMITTED**

### Responses
The JSON representation of `response` object is made of these parts:
- mandatory `id` member : the request id this response corresponds to
- optional `error` member : whether an error occurred during the parsing of the method or during it's execution this member **MUST** be present and valued. If no errors occurred this member **MUST NOT** be present. For a detailed structure of the `error` member see below.
- optional `result` member : This **MUST** be set, if the corresponding request requires a result from the user. If no errors occurred by invoking the corresponding function, this member **MUST** be present even if one or more informations are null. The type can be of Object or single type Array or Primitive string/number. If no data is meant back for the issuer (the method is void on the receiver) or an error occurred this member **MUST NOT** be present.

Notice some differences with standard JSON-RPC-2.0. Namely the result member is not always required. A response like this:
```json
{"id": 2}
```
means "request received and processed correctly with no data to send back".

To better clarify the concept and clear the field of free interpretations let's take another example of **wrong response**
```json
{"id": 2, "result": false}
```
This response syntax leaves room to many interpretations or questions: is it an error? is `false` the legit response value to the issued request?

We reiterate,  responses **MUST BE** of two types:
- success responses : no error occurred during the processing, the request was legit, syntactically correct, and the receiver had no issues processing it. This kind of response **MUST NOT** have the `error` member and **MAY** have the `result` member if a value is expected back to the issuer.
- failure responses : something wrong with the request, its syntax, its validity scope, or server processing problems. This kind of response **MUST HAVE** the `error` member and **MAY** have the `result` member.

The latter deserves a better explanation: failure responses can be distinguished by a severity degree. 
Example 1 : a client submits a solution and server rejects it cause it's not below target. Server **MUST** respond like this
```json
{
  "id": 1,
  "error": {
      "code": 406,
      "message" : "Bad nonce"
  }
}
```
Example 2 : a client submits a solution and server **accepts** it **but** it accounts the share as stale. Server **MUST** respond like this
```json
{
  "id": 1,
  "error": {
      "code": 202,
      "message" : "Stale"
  }
}
```
Example 3 : a client submits an authorization request specifying an invalid workername. Server authorizes the account but rejects worker name. Server **MUST** respond like this
```json
{
  "id": 1,
  "error": {
      "code": 215,
      "message" : "Invalid Worker Name"
  }
}
```

Example 1 depicts the condition of a severe failure while Example 2 and 3 depict a situation where the request has been accepted and processed properly but the result **MAY NOT** be what expected by the client.
It's up to the client to evaluate the severity of the error and decide whether to proceed or not.

Use proper error codes so that pools may inform miners about the condition of their requests. Error codes **MUST** honor this scheme:

- Error codes 2xx : request accepted and processed but some additional info in the `error` member may give hint
- Error codes 3xx : server could not process the request due to a lack of authorization by the client
- Error codes 4xx : server could not process the request due to syntactic problems (method not found, missing params, wrong data types ecc.) or passed `param` values are not valid.
- Error codes 5xx : server could not process the request due to internal errors

### Notifications
A notification message has the same representation of a `request` with the only difference being that the `id` member **MUST NOT** be present. This means the issuer is not interested nor expects any response to this message. It's up to the receiver to take actions accordingly. For instance, the receiver **MAY** decide to execute the method, or, in case of errors or methods not allowed, drop the connection and close the session.

#### Error member
As seen above a `response` **MAY** contain an `error` member. When present this member **MUST** be an Object with:
- mandatory member `code` : a Number which identifies the error occurred
- mandatory member `message` : a short human readable sentence describing the error occurred
- optional member `data` : a Structured or Primitive value that contains details about the error. The value of this member is defined by the Server (e.g. detailed error information, nested errors, etc.).

## Protocol Flow
- Client starts session by opening a TCP socket to the server
- Client advertises and request protocol compatibility
- Server confirms compatibility and declares ready
- Client starts/resumes a session
- Client sends request for authorization for each of its workers
- Server replies back with responses for each authorization
- Server sends `mining.set` for constant values to be adopted for following mining jobs
- Server sends `mining.notify` with minimal job info
- Client mines on job
- Client sends `mining.submit` if any solution is found for the job
- Server replies whether solution is accepted
- Server optionally sends `mining.set` for constant values to be adopted for following mining jobs (if something changed)
- Server sends `mining.notify` with minimal job info
- ... (continue)
- Eventually either party closes session and TCP connection

### Session Handling - Hello
~~One of the worst annoyances until now is that server, at the very moment of socket connection, does not provide any useful information about the stratum flavour implemented. This means the client has to start a conversation by iteratively trying to connect via different protocol flavours. This proposal amends the situation making mandatory for the server to advertise itself to the client. 
When a new client connects to the server, the server **MUST** send a `mining.hello` notification :~~

It has been noted that giving the server the duty to advertise itself as the first message of the conversation could potentially be harmful in respect of traffic amplification attacks using spoofed IP addresses or in traditional DDoS attacks where an attacker needs to spend very little resources to force the server to send a large packet back.
For this reason the duty of the first advertisement is kept on client which will issue a `mining.hello` request like this:

```json
{
  "id" : 0,
  "method": "mining.hello", 
  "params": 
  { 
    "agent": "ethminer-0.17",
    "host" : "somemininigpool.com",
    "port" : "4d2",
    "proto": "EthereumStratum/2.0.0"
  }
}
```
The `params` member object has these mandatory members:
- `agent` (string) the mining software version
- `host` (string) the host name of the server this client is connecting to
- `port` (hex) the port number of the server this client is connecting to
- `proto` (string) the stratum flavour requested and expected to be implemented by the server

Sending host and port enables virtual hosting, where virtual pools or private URLs might be used for DDoS protection, but that are aggregated on Stratum server backends. As with HTTP, the server **CANNOT** trust the host string. The port is included separately to parallel the `client.reconnect` method (see below).

If the server is prepared to start/resume a session with such requirements it **MUST** reply back with a response like this:
```json
{
  "id" : 0,
  "result": 
  { 
    "proto" : "EthereumStratum/2.0.0",
    "encoding" : "plain",
    "resume" : "1",
    "timeout" : "b4",
    "maxerrors" : "5",
    "node" : "Geth/v1.8.18-unstable-f08f596a/linux-amd64/go1.10.4"
  } 
}
```
Where the `result` is an object made of 5 mandatory members
- `proto` (string) which **MUST** match the exact version requested by the client
- `encoding` (string) states whether or not all **next messages** should be read as plain text or as streams of data. Actually the only possible value is or "plain" (Gzip compression is dropped as it's not efficient on such small sized messages)
- `resume` (hex bool) states whether or not the host can resume a previously created session;
- `timeout` (hex) the number of seconds after which the server is allowed to drop connection if no messages are received from the client
- `maxerrors` (hex) the maximum number of errors the server will bear before abruptly closing connection
- `node` (string) the node software version underlying the pool

~~When the server replies back with `"encoding" : "gzip"` to the client, both parties **MUST** gzip compress all next messages. In case the client is not capable of compression it **MUST** close the connection immediately.
Should the server, after this reply, receive other messages as plain text, it **MUST** close the connection.~~

Eventually the client will continue with `mining.subscribe` (described below)

In case of errors or rejection to start the conversation, the server **MAY** reply back with an error giving the other party useful information or, at server's maintainers discretion, abruptly close the connection.
```json
{
  "id" : 0,
  "error": 
  { 
      "code": 400,
      "message" : "Bad protocol request"
  } 
}
```
or 
```json
{
  "id" : 0,
  "error": 
  { 
      "code": 403,
      "message" : "Forbidden - Banned IP address"
  } 
}
```
_The above two JSON error values are only samples_

Eventually the server will close the connection.

Why should a pool advertise the node's version? It's a matter of transparency. Miners should know whether or not the pool has upgraded to latest patches/releases for node's software.

### Session Handling - Bye
Disconnection is not gracefully handled in Stratum. Client disconnections from pool may be due to several errors. This leads to wasted TCP sockets on the server, which wait for keepalive timeouts to trigger. A useful notification is `mining.bye` which, once processed, allows both parties of the session to gracefully close TCP connections.
```json
{
  "method": "mining.bye"
}
```
The party receiving this message acknowledges the other party wants to stop the conversation and closes the socket. The issuer will close too. The explicit issuance of this notification implies the session gets abandoned. No session resuming will be possible even on a server which supports session resuming. Clients reconnecting to the same server which implements session resuming **SHOULD** expect a new session id and **MUST** re-authorize all their workers.

### Session Handling - Session Subscription
After receiving the response to `mining.hello` from the server, the client, in case the server does support session resuming, **MAY** request to resume a previously interrupted session with `mining.subscribe` request:
```json
{
  "id": 1,
  "method": "mining.subscribe", 
  "params": "s-12345"
}
```
where `params` is the id of the session the client wants to resume.

Otherwise, if client wants to start a new session **OR** server does not support session resuming, the request of subscription **MUST** omit the `params` member:
```json
{
  "id": 1,
  "method": "mining.subscribe"
}
```

### Session Handling - Response to Subscription
A server receiving a client session subscription **MUST** reply back with
```json
{
  "id": 1,
  "result": "s-12345"
}
```
A server receiving a subscription request with `params` being a string holding the session id. This cases may apply
- If session resuming is not supported `result` will hold a new session Id which **MUST** be a different value from the `session` member issued by client in previous `mining.subscribe` method 
- If session resuming is supported the server will retrieve working values from cache and `result` will have the same id requested by the client. This means a session is "resumed": as a consequence the server **CAN** start pushing jobs omitting to precede them with `mining.set` (see below) and the client **MUST** continue to use values last received within the same session scope. In addition the client **CAN** omit to re-authorize all its workers.
- If session resuming is supported but the requested session cannot be found for any reason (expiration, cache invalidation), `result` will hold a new session Id which **MUST** be a different value from the `session` member issued by client in previous `mining.subscribe` method. As a consequence the server **MUST** wait for the client to request authorization for its workers and **MUST** send `mining.set` values before pushing jobs. The client **MUST** prepare for a new session discarding all previously cached values (if any).

A server implementing session-resuming **MUST** cache:
- Session Ids
- Any authorized worker
- Any active job per session
- ExtraNonce

Servers **MAY** drop entries from the cache on their own schedule. It's up to the server to enforce session validation for same agent and/or ip.

A client which successfully subscribes and resumes session (the `session` value in server response is identical to `session` value requested by client on `mining.subscribe`) **CAN** omit to issue the authorization request for its workers.

### Session Handling - Noop
There are cases when a miner struggles to find a solution in a reasonable time so it may trigger the timeout imposed by the server in case of no communications (the server may think the client got disconnected). To mitigate the problem a new method `mining.noop`(with no additional parameters) may be sent by the client. 
```json
{
  "id": 1,
  "method": "mining.noop"
}
```

### Session Handling - Reconnect
Under certain circumstances the server may need to free some resources and or relocate miners to another machine. Until now the only option for servers was to abruptly close the connection. On the miner's side this action is interpreted as a server malfunction and they, more often than not, switch to a fail-over pool. 
The implementation of the notification `mining.reconnect` helps client to better merge with logic of handling of large mining pools. 
```json
{
  "method": "mining.reconnect",
  "params": {
      "host": "someotherhost.com",
      "port": "d80",
      "resume": "1"
  }
}
```
This notification is meant only from servers to clients. Should a server receive such a notification it **MUST** ignore it. After the notification has been properly sent, the server is ALLOWED to close the connection, while the client will take the proper actions to reconnect to the suggested end-point.
The `host` member in `params` object **SHOULD** report an host DNS name and not an IP address: TLS encrypted connections validate the CN name in the certificate which is most generally, a host name. 
The third member `resume` (hex bool) of the `params` object sets whether or not the receiving server is prepared for session resuming.
After this notification has been issued by the server, the client should expect no further messages and **MUST** disconnect.

### Workers Authorization
The miner **MUST** authorize at least one worker in order to begin receiving jobs and submit solutions or hashrates. The miner **MAY** authorize multiple workers in the same session. The server **MUST** allow authorization for multiple workers within a session and **MUST** validate at least one authorization from the client before starting to send jobs. A `worker` is a tuple of the address where rewards must be credited coupled with identifier of the machine actually doing the work. For Ethereum the most common form is `<account>.<MachineName>`. The same account can be bound to multiple machines. For pools allowing anonymous mining the account is the address where rewards must be credited, while, for pools requiring registration, the account is the login name. Each time a solution is submitted by the client it must be labelled with the Worker identifier. It's up to the server to keep correct accounting for different addresses.

The syntax for the authorization request is the following:
```json
{
  "id": 1,
  "method": "mining.authorize", 
  "params": ["<account>[.<MachineName>]", "password"]
}
```
`params` member must be an Array of 2 string elements. For anonymous mining the "password" can be any string value or empty but not null. Pools allowing anonymous mining will simply ignore the value.
The server **MUST** reply back either with an error or, in case of success, with
```json
{
  "id": 1,
  "result": "w-123"
}
```
Where the `result` member is a string which holds an unique - within the scope of the `session` - token which identifies the authorized worker. For every further request issued by the client, and related to a Worker action, the client **MUST** use the token given by the server in response to an `mining.authorize` request. This reduces the number of bytes transferred for solution and/or hashrate submission.

If the client is resuming a previous session it **CAN** omit the authorization request for its workers and, in this case, **MUST** use the tokens assigned in the originating session. It's up to the server to keep the correct map between tokens and workers.
The server receiving an authorization request where the credentials match previously authorized ones within the same session **MUST** reply back with the previously generated unique token.

### Prepare for mining
A lot of data is sent over the wire multiple times with useless redundancy. For instance, the seed hash is meant to change only every 30000 blocks (roughly 5 days) while fixed-diff pools rarely change the work target. Moreover pools must optimize search segments among miners trying to assign to every session a different "startNonce" (AKA extraNonce).
For this purpose the `notification` method `mining.set` allows the server to set on the client, only those params which change less frequently. The server will keep track of seed, target and extraNonce at session level and will push a notification `mining.set` whenever any (or all) of those values change for the connected miner.
```json
{
  "id": 1,
  "method": "mining.set", 
  "params": {
      "epoch" : "dc",
      "target" : "0112e0be826d694b2e62d01511f12a6061fbaec8bc02357593e70e52ba",
      "algo" : "ethash",
      "extranonce" : "af4c"
  }
}
```
At the beginning of each `session` the server **MUST** send this notification before any `mining.notify`. All values passed by this notification will be valid for all **NEXT** jobs until a new `mining.set` notification overwrites them. Description of members is as follows:
- optional `epoch` (hex) : unlike all actual Stratum implementations the server should inform the client of the epoch number instead of passing the seed hash. This is enforced by two reasons: the first one is that the client has only one way to compute the epoch number and this is by a linear search from epoch 0 iteratively trying increasing epochs till the hash matches the seed hash. Second reason is that epoch number is more concise than seed hash. In the end, the seed hash is only transmitted to inform the client about the epoch and is not involved in the mining algorithm.
- optional `target` (hex) : this is the boundary hash already adjusted for pool difficulty. Unlike in EthereumStratum/1.0.0, which provides a `mining.set_difficulty` notification of an _index of difficulty_, the proponent opt to pass directly the boundary hash. If omitted the client **MUST** assume a boundary of `"0x00000000ffff0000000000000000000000000000000000000000000000000000"`
- optional `algo` (string) : the algorithm the miner is expected to mine on. If omitted the client **MUST** assume `"algo": "ethash"`
- optional `extranonce` (hex) : a starting search segment nonce assigned by the server to clients so they possibly do not overlap their search segments. If server wants to "cancel" a previously set extranonce it must pass this member value as an empty string.

Whenever the server detects that any of the four values change within the session, the server will issue a notification with only the changed members in the `param` object. For this reason on each **new** session the server **MUST** pass all four members. As a consequence the miner is instructed to adapt those values on the **next** job which gets notified.

The new `algo` member is defined to be prepared for possible presence of algorithm variants to ethash, namely ethash1a or ProgPow.

Pools providing multicoin switching will take care to send a new `mining.set` to miners before pushing any job after a switch.

The client which can't support the data provided in the `mining.set` notification **MAY** close connection or stay idle till new values satisfy its configuration (see `mining.noop`).

All client implementations **MUST** be prepared to accept new extranonces during the session: unlike in EthereumStratum/1.0.0 the optional client advertisement `mining.extranonce.subscribe` is now implicit and mandatory.

The miner receiving the `extranonce` **MUST** initialize the search segment for next job resizing the extranonce to a hex of 16 bytes thus appending as many zeroes as needed.
Extranonce "af4c" means "_search segment of next jobs starts from 0xaf4c000000000000_"

If `extranonce` is valued to an empty string, or its never been set within the session scope, the client is free to pick any starting point of its own search segment on subsequent `mining.notify` jobs.

### Details of "extranonce"

Miners connected to a pool might likely process the very same nonces thus creating a lot of wasteful duplicate jobs. A `nonce` is any valid number which, applied to algorithm and job specifications, produces a result which is below a certain target. For every job pushed by server to client(s) there are 2^64 possible nonces to test.

Note that:
- Any nonce in the 2^64 has the very same possibility to be the right one. 
- A hashing job can be solved by more than one nonce

Every "test" over a number is called a hash. Assuming a miner should receive a job for each block and considering the actual average block time of 15 seconds that would mean a miner should try

```
  ( 2^64 / 15 ) / 1T ~ 1,229,782.94 TeraHashes per second
```

This computation capacity is well beyond any miner on the market (including ASICs). For this reason single miners can process only small chunks (segments) of this humongous range. The way miners pick the segments to search on is beyond the scope of this work. As miners are not coordinated, there is no knowledge - for a single miner - of segments picked by other miners.
Extranonce concept is here to mitigate this possibility of duplicate jobs. Charging the server (the work provider) to give miners, at the maximum possible extent, different segments to search on.

Giving the above assumptions we can depict a nonce as any number in the hex range:

```
  Min 0x0000000000000000
  Max 0xffffffffffffffff
```
_the prefix 0x is voluntarily inserted here only to give a better visual representation_.

The `extranonce` is, at its basics, the message from the server saying to the client "_I give you the first number to start search from_". The `extranoce` is the leftmost part of that number.
Assume a pool notifies the client of extranonce `ab5d`. This means the client will see its search segment narrowed as 
```
  Min 0xab5d000000000000
  Max 0xab5dffffffffffff
```
Pushing an extranonce of 4 bytes (like in the example) will give pool the ability to segment 65535 different miners (or if you prefer 0xffff miners) while leaving the miner a segment of 2^48 possible nonces to search on.

The computation capacity needed to search this segment is
```
  ( 2^48 / 15 ) / 1T ~ 18.76 TeraHashes per second
```
Which is still a wide segment where miners can randomly (or using other ergodic techniques) pick their internal search segments.

Extranonce **MUST** be passed with all relevant bytes (no omission of left zeroes) for a specific reason. Assume an extranonce of "01ac" : it  has the same decimal value of "1ac" but the number of bytes changes thus changing available search segment

```
  When "01ac"               When "1ac"
  Segment is                Segment is
  Min  0x01ac000000000000   Min  0x1ac0000000000000
  Max  0x01acffffffffffff   Max  0x1acfffffffffffff
```
As you can see resulting segments are quite different

This all said pools (server), when making use of extranonce, **MUST** observe a maximum length of 6 bytes (hex).

### Jobs notification
When available server will dispatch jobs to connected miners issuing a `mining.notify` notification.
```json
{
  "id": 1,
  "method": "mining.notify", 
  "params": [
      "bf0488aa",
      "6526d5",
      "645cf20198c2f3861e947d4f67e3ab63b7b2e24dcc9095bd9123e7b33371f6cc",
      "0"
  ]
}
```
`params` member is made of 4 mandatory elements:
- 1st element is jobId as specified by pool. To reduce the amount of data sent over the wire, pools **SHOULD** keep their job IDs as concise as possible. Pushing a Job id which is identical to headerhash is a bad practice and is highly discouraged
- 2nd element is the hex number of the block id
- 3rd element is the headerhash
- 4th element is a hex boolean indicating whether or not eventually found shares from previous jobs will be accounted for as stale

### Solution submission
When a miner finds a solution for a job, it sends a `mining.submit` request to server.
```json
{
  "id": 1,
  "method": "mining.submit", 
  "params": [
      "bf0488aa",
      "68765fccd712",
      "w-123"
  ]
}
```
First element of `params` array is the jobId this solution refers to (as sent in the `mining.notify` message from the server). Second element is the `miner nonce` as hex. Third element is the token given to the worker previous `mining.authorize` request. Any `mining.submit` request bound to a worker which was not succesfully authorized - i.e. the token does not exist in the session - **MUST** be rejected.

You'll notice in the sample above the `miner nonce` is only 12 bytes wide (should be 16). Why?
That's because in the previous `mining.set` the server has set an `extranonce` of `af4c`. This means the full nonce is `af4c68765fccd712`
In presence of extranonce the miner **MUST** submit only the chars to append to the extranonce to build the final hex value. If no extranonce is set for the session or for the work the miner **MUST** send all 16 bytes.

It's the server duty to keep track of the tuples `job ids <-> extranonces` per session.

When the server receives this request it either responds success using the short form
```json
{"id": 1}
```
or, in case of any error or condition with a detailed error object
```json
{
  "id": 1,
  "error": {
      "code": 404,
      "message" : "Job not found"
  }
}
```

Client **should** treat errors as "soft" errors (stales) or "hard" (bad nonce computation, job not found, etc.). Errors in 5xx range are server errors and suggest that the miner abandon the connection and switch to a fail-over.

### Hashrate
Most pools offer statistic information, in form of graphs or by API calls, about the calculated hashrate expressed by the miner. Miners like to compare this data with the hashrate they read on their devices. This information has never been coded in Stratum and most pools adopt the method from getWork named `eth_submitHashrate`.

In this document we propose an official implementation of the `mining.hashrate` request. This method behaves differently when issued from client or from server.

#### Client sends its hashrate to the server
```json
{
  "id" : 1,
  "method": "mining.hashrate",
  "params": [
      "500000",
      "w-123"
      ]
}
```
where `params` is an array made of two elements: the first is a hexadecimal string representation (32 bytes) of the hashrate the miner reads on its devices. the latter is the authorization token issued to worker this hashrate refers to (see above for `mining.authorization`).

Server **MUST** respond back with either an acknowledgment message
```json
{"id": 1}
```
Optionally the server can reply back reporting its findings about calculated hashrate **for the same worker**.
```json
{
  "id": 1,
  "result" : [
      "4f0000",
      "w-123"
      ]
}
```
In case of errors - for example when the client submits too frequently - with
```json
{
  "id": 16,
  "error" : {
    "code": 220,
    "message": "Enhance your calm. Too many requests"
  }
}
```

#### Server communicates hashrate to client
Optionally the server can **notify** client about its overall performance (according to the schedule set on server) with a `mining.hashrate` notification composed like this
```json
{
  "method": "mining.hashrate",
  "params": {
      "interval": 60,
      "hr": "500000",
      "accepted": [3692,20],
      "rejected": 0,
  }
}
```
Where `params` is an object which holds these members for values of the **whole session**:
- `interval` (number) the width, in minutes, of the observation window. "_in the last x minutes we calculated ..._"
- `hr` (hex) representation of the hashrate the pool has calculated for the miner
- `accepted` is an array of two number elements : the first is the overall count of accepted shares. the second is the number of stale shares. The array must be interpreted as "total accepted of which x are stale"
- `rejected` (number) the overall number of rejected shares

The client can use this information to take internal actions to reset/restart its workers.
