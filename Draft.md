# EthereumStratum/2.0.0
## Authors
- Andrea Lanfranchi
- Marius Van Der Wijden
- Paweł Bylica
- Goobur
## Abstract
This draft contains the guidelines to define a new standard for the Stratum protocol used by Ethereum miners to communicate with mining pool servers.
### Conventions
The key words `MUST`, `MUST NOT`, `REQUIRED`, `SHALL`, `SHALL NOT`, `SHOULD`, `SHOULD NOT`, `RECOMMENDED`, `MAY`, and `OPTIONAL` in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).
The definition `mining pool server`, and it's plural form, is to be interpreted as `work provider` and later in this document can be shortened as `pool` or `server`.
The definition `miner(s)`, and it's plural form, is to be interpreted as `work receiver/processor` and later in this document can be shortened as `miner` or `client`.
### Rationale
Ethereum does not have an official Stratum implementation yet. It officially supports only getWork which requires miners to constantly pool the work provider. Only recently go-ethereum have implemented a [push mechanism](https://github.com/ethereum/go-ethereum/pull/17347) to notify clients for mining work, but whereas the vast majority of miners do not run a node, it's main purpose is to facilitate mining pools rather than miners.
The Stratum protocol on the other hand relies on a standard stateful TCP connection which allows two-way exchange of line-based messages. Each line contains the string representation of a JSON object following the rules of either [JSON-RPC 1.0](https://www.jsonrpc.org/specification_v1) or [JSON-RPC 2.0](https://www.jsonrpc.org/specification).
Unfortunately, in absence of a well defined standard, various flavours of Stratum have bloomed for Ethereum mining as a derivative work for different mining pools implementations. The only attempt to define a standard was made by NiceHash with their [EthereumStratum/1.0.0](https://github.com/nicehash/Specifications/blob/master/EthereumStratum_NiceHash_v1.0.0.txt) implementation which is the main source this work inspires from.
Mining activity, thus the interaction among pools and miners, is at it's basics very simple, and can be summarized with "_please find a number (nonce) which coupled to this data as input for a given hashing algorithm produces, as output, a result which is below a certain target_". Other messages which may or have to be exchanged among parties during a session are needed to support this basic concept.
Due to the simplicity of the subject, the proponent, means to stick with JSON formatted objects rather than investigating more verbose solutions, like for example  [Google's Protocol Buffers](https://developers.google.com/protocol-buffers/docs/overview) which carry the load of strict object definition.
### Stratum design flaws
The main Stratum design flaw is the absence of a well defined standard. This implies that miners (and mining software developers) have to struggle with different flavours which make their life hard when switching from one pool to another or even when trying to "guess" which is the flavour implemented by a single pool. Moreover all implementations still suffer from an excessive verbosity for a chain with a very small block time like Ethereum. A few numbers may help understand. A normal `mining.notify` message weigh roughly 240 bytes: assuming the dispatch of 1 work per block to an audience of 50k connected TCP sockets means the transmission of roughly 1.88TB of data a month. And this can be an issue for large pools. But if we see the same figures the other way round, from a miner's perspective, we totally understand how mining decentralization is heavily affected by the quality of internet connections.
### Sources of inspiration
- [NiceHash EthereumStratum/1.0.0](https://github.com/nicehash/Specifications/blob/master/EthereumStratum_NiceHash_v1.0.0.txt)
- [Zcash variant of the Stratum protocol](https://github.com/zcash/zips/blob/23d74b0373c824dd51c7854c0e3ea22489ba1b76/drafts/str4d-stratum/draft1.rst#json-rpc-1-0)
- [BetterHash Mining Protocol](https://github.com/TheBlueMatt/bips/blob/master/bip-XXXX.mediawiki)

## Specification
The Stratum protocol is an instance of [JSON-RPC-2.0](https://www.jsonrpc.org/specification). The miner is a JSON-RPC client, and the server is a JSON-RPC server. All communications exist within the scope of a `session`. A session starts at the moment a client opens a TCP connection to the server till the moment either party do voluntary close the very same connection or it gets broken. Servers **MAY** support session resuming if this is initially negotiated (on first session handshaking) between the client and the server. During a session all messages exchanged among server and client are line-based which means all mesasges a JSON strings terminated by ASCII LF character (which may also be denoted as `\n` in this document). The LF character **MUST NOT** appear elsewhere in a message. Client and server implementations **MUST** assume that once they read a LF character, the current message has been completely received and can be processed.
Line messages are of three types :
- `Requests` : messages for which the issuer expects a response. The receiver **MUST** reply back to any request individually
- `Responses` : solicited messages by a previous request. The responder **MUST** label the response with the same identifier of the originating request.
- `Notifications` : unsolicited messages for which the issuer is not interested nor is expecting a response. Nevertheless a response (eg. an aknowledgement) **MAY** be sent by the receiver.

During a `session` both party **CAN** exchange messages of the above depicted three types.

As per [JSON-RPC-2.0](https://www.jsonrpc.org/specification) specification requests and responses differ from notifications by the identifier (`id`) member in the JSON object: 
- Requests **MUST** have an `id` member
- Responses **MUST** have an `id` member valued exactly as the `id` member of the request this response is for
- Notifications **MUST NOT** have an `id` member

Unlike standard JSON-RPC-2.0 in EthereumStratum/2.0.0 the `id` member **MUST NOT** be `null`. If the member is present (requests or responses) it **MUST** be valued to an integer Number ranging from 0 to 65535. Please note that a message with `"id": 0` **MUST NOT** be interpreted as a notification. The removal of other identifier types (strings) is due to the need to reduce the number of bytes transferred.

### Requests
The JSON representation of `request` object is made of these parts:
- mandatory `id` member of type Integer : the identifier established by the issuer
- mandatory `jsonrpc` member of type String : constantly valued to string "2.0"
- mandatory `method` member of type String : the name of the method to be invoked on the receiver side
- optional `params` member : in case the method invocation on the receiver side requires the application of additional parameters to be executed. The type **CAN** be Object (with named members of different types) or Array of single type. In case of an array the parameters will be applied by their ordinal position. If the method requested for invocation on the receiver side does not require the application of additional parameters this member **MUST NOT** be present. The notation `"params" : null` **IS NOT PERMITTED**

### Responses
The JSON representation of `response` object is made of these parts:
- mandatory `id` member of type Integer : the identifier of the request this response corresponds to
- mandatory `jsonrpc` member of type String : constantly valued to string "2.0"
- optional `error` member : whether an error occurred during the parsing of the method or during it's execution this member **MUST** be present and valued. If no errors occurred this member **MUST NOT** be present. For a detailed structure of the `error` member see below.
- optional `result` member : This has to be set, if the corresponding request requires a result from the user. If no errors  occured by invoking the corresponding function, this member **MUST** be present even if one or more informations are null. The type can be of Object or single type Array. If no data is meant back for the issuer (the method is void on the receiver) or an error occurred this member **MUST NOT** be present.

You'll notice here some differences with standard JSON-RPC-2.0. Namely the result member is not always required. Basically a response like this :
```json
{"id": 2, "jsonrpc": "2.0"}
```
means "request received and processed correctly with no data to send back".

### Notifications
A notification message has the very same representation of a `request` with the only difference the `id` member **MUST NOT** be present. This means the issuer is not interested nor expects any reponse to this message. It's up to the receiver to take actions accordingly. For instance the receiver **MAY** decide to execute the method, or, in case of errors or methods not allowed, drop the connection thus closing the session.

#### Error member
As seen above a `response` **MAY** contain an `error` member. When present this member **MUST** be Object with:
- mandatory member `code` : a Number which identifies the error occurred
- mandatory member `message` : a short human readable sentence describing the error occurred
- optional member `data` : a Structured or Primitive value that contains additional informations about the error. The value of this member is defined by the Server (e.g. detailed error information, nested errors etc.).

## Protocol Flow
- Client starts session by opening a TCP socket to the server
- Server advertises it's capabilities
- Client sends request for authorization for each of it's workers
- Server replies back with responses for each authorization
- Server sends `mining.set` for constant values to be adopted for following mining jobs
- Server sends `mining.notify` with minimal job info
- Client mines on job
- Client sends `mining.submit` if any solution found for the job
- Server replies wether solution is accepted
- Server optionally sends `mining.set` for constant values to be adopted for following mining jobs (if something changed)
- Server sends `mining.notify` with minimal job info
- ... (continue)
- Eventually either party closes session and TCP connection

### Session Handling - Hello
One of the worst annoyances until now is that server, at the very moment of socket connection, does not provide any useful information about the stratum flavour implemented. This means the client has to start a conversation by iteratively trying to connect via different protocol flavours. This proposal amends the situation making mandatory for the server to advertise itself to the client. 
When a new client to the server, the server **MUST** send a `mining.hello` notification :
```json
{
  "jsonrpc": "2.0",
  "method": "mining.hello", 
  "params": 
  { 
      "proto": "EthereumStratum/2.0.0",
      "resume" : true,
      "timeout" : 180,
      "maxerrors" : 5
  } 
}
```
The `params` member object has these mandatory members:
- `proto` (string) which reports the stratum flavour implemented by the server;
- `resume` (bool) which states whether or not the host can resume a previously created session;
- `timeout` which reports the number of seconds after which the server is allowed to drop connection if no messages from the client
- `maxerrors` the maximum number of errors the server will bear before abruptly close connection

The client receiving this message will decide whether or not it's software is compatible with the protocol implementation and eventually continue with the conversation.

### Session Handling - Bye
Disconnection are not gracefully handled in Stratum. Client disconnections from pool may be due to several errors and this leads to waste of TCP sockets on server's side which wait for keepalive timeouts to trigger. A useful notification is `mining.bye` which, once processed, allows both parties of the session to stop receiving and gracefully close TCP connections
```json
{
  "method": "mining.bye"
}
```
The party receiving this message aknowledges the other party wants to stop the conversation and closes the socket. The issuer will close too.

### Session Handling - Subscription
After receiving the `mining.hello` from server, the client **MUST** advertise itself with `mining.subscribe` request, if their willing to proceed:
```json
{
  "id": 1,
  "jsonrpc": "2.0",  
  "method": "mining.subscribe", 
  "params": 
  { 
      "agent": "ethminer-0.17",
      "host" : "somemininigpool.com",
      "port" : 1234,
      "session" : null
  } 
}
```
where members are :
- `agent` (string) the mining software version
- `host` (string) the host name of the server this client is willing to connect to
- `port` (number) the port number of the server this client is willing to connect to
- `session` (string) the session id previously provided by the server in case the server supports session resuming

In case server does not support session resuming the server **MUST** ignore the member. If the client detects (from previous mining.hello) that server does not support session resuming, it **CAN** omit the member or value it to `null`
If client connects to a server which implements session resuming but want to start a new session either omits the `session` member or values it to `null`

The rationale behind sending host and port is it enables virtual hosting, where virtual pools or private URLs might be used for DDoS protection, but that are aggregated on Stratum server backends. As with HTTP, the server CANNOT trust the host string. The port is included separately to parallel the client.reconnect method (see below).

### Session Handling - Response to Subscription
A server receiving a client session subscription will reply back with
```json
{
  "id": 1,
  "jsonrpc": "2.0",  
  "result": 
  { 
      "subscribed": true,
      "session" : "abcdefgh123456"
  } 
}
```
A server receiving a subscription request with parameter's member `session` valued will apply this logic:
- If session resuming is not supported it will value `session` with a new id
- If session resuming is supported it will retrieve working values from cache and will value `session` with the same id requested by the client
- If session resuming is supported but the requested session has expired or it's cache values have been purged then it responds with `session` valued to a new id

A server implementing session-resuming **MUST** cache :
- The session Ids
- Any active job per session
- The extraNonce
- Any authorized worker

Servers **MAY** drop entries from the cache on their own schedule. It's up to server to enforce session validation for same agent and/or ip.

A client which successfully subscribes and resumes session (the `session` value in server response is identical to `session` value requested by client on `mining.subscribe`) **CAN** omit to issue the authorization request for it's workers.

### Session Handling - Noop
There are cases when a miner struggles to find a solution in a reasonable time so it may trigger the timeout imposed by the server in case of no communications (the server, in fact, may think the client got disconnected). To mitigate the problem a new method `mining.noop`(with no additional parameters) may be requested by the client. 
```json
{
  "id": 50,
  "jsonrpc": "2.0",
  "method": "mining.noop"
}
```

### Workers Authorization
A miner **MUST** authorize at least one worker in order to submit solutions. A miner **MAY** authorize multiple workers in the same session. A server **MUST** allow authorization for multiple workers within a session. A `worker` is a tuple of the address where rewards must be credited coupled with identifier of the machine actually doing the work. For Ethereum the most common form is `<account>.<MachineName>`. The same account can be bound to multiple machines. For pool's allowing anonymous mining the account is the address where rewards must be credited, while, for pools requiring registration, the account is the login name. Each time a solution is submitted by the client it must be labelled with the Worker identifier. It's up to server to keep the correct accounting for different addresses.

The syntax for the authorization request is the following:
```json
{
  "id": 2,
  "jsonrpc": "2.0",  
  "method": "mining.authorize", 
  "params": ["<account>[.<MachineName>]", "password"]
}
```
`params` member must be an Array of 2 string elements. For anonymous mining the "password" can be any string value or empty but not null. Pools allowing anonymous mining will simply ignore the value.
### Prepare for mining
A lot of data is sent over the wire multiple times with useless redundancy. For instance the seed hash is meant to change only every 30000 blocks (roughly 5 days) while fixed-diff pools rarely change the work target. Moreover pools must optimize the search segments among miners trying to assign to every session a different "startNonce" (AKA extraNonce).
For this purpose the `notification` method `mining.set` allows to set (on miner's side) only those params which change less frequently. The server will keep track of seed, target and extraNonce at session level and will push a notification `mining.set` whenever any (or all) of those values changes to the connected miner.
```json
{
  "jsonrpc": "2.0",
  "method": "mining.set", 
  "params": {
      "algo" : "ethash",
      "extranonce" : "0xaf4c",
      "seed" : "0xa7958ab88d27488e1b03ec3d1aeca336d8baffd356f32fdf00d3772b24765d28",
      "target" : "0x0112e0be826d694b2e62d01511f12a6061fbaec8bc02357593e70e52ba"
  }
}
```
Whenever the server detects that one, or two, or three or four values change within the session, the server will issue a notification with one, or two or three or four members in the `param` object. As a consequence the miner is instructed to adapt those values on **next** job which gets notified.
The new `algo` member is defined to be prepared for possible presence of algorithm variants to ethash, namely ethash1a or ProgPow.
Pools providing multicoin switching will take care to send a new `mining.set` to miners before pushing any job after a switch.
The client wich can't support the data provided in the `mining.set` notification **MAY** close connection or stay idle till new values satisfy it's configuration (see `mining.noop`)

### Jobs notification
When available server will dispatch jobs to connected miners issuing a `mining.notify` notification.
```json
{
  "jsonrpc": "2.0",
  "method": "mining.notify", 
  "params": [
      "bf0488aa",
      "0x645cf20198c2f3861e947d4f67e3ab63b7b2e24dcc9095bd9123e7b33371f6cc",
      false
  ],
  "block" : "0x64b577"
}
```
First element of `params` array is jobId as specified by pool. To reduce the amount of data sent over the wire pools **SHOULD** keep their job IDs as concise as possible. Pushing a Job id which is identical to headerhash is a bad practice.
The second element of `params` array is the headerhash. The third element is an optional boolean indicating whether or not eventually found shares from previous jobs will be accounted for sure as stale.
Optionally the server can push also the member `block` to inform the miner on the actual height of the chain he is mining on. The information would be useful for miners in case they need to async prepare the workers for an epoch/DAG change.

### Solution submission
When a miner finds a solution for a job he is mining on it sends a `mining.submit` request to server.
```json
{
  "id": 31,
  "jsonrpc": "2.0",
  "method": "mining.submit", 
  "params": [
      "bf0488aa",
      "0xa60b68765fccd712",
      "<account>[.<MachineName>]"
  ]
}
```
First element of `params` array is the jobId this solution refers to (as sent in the `mining.notify` message from the server). Second element is the found nonce. Third element is the fully qualified worker name as registered in previous `mining.authorize` request. Any `mining.submit` request bound to a worker which was not succesfully authorized should be rejected.

When the server receives this request it either responds success using the short form
```json
{"id": 31, "jsonrpc": "2.0"}
```
or, in case of any error or condition with a detailed error object
```json
{
  "id": 31,
  "jsonrpc": "2.0",
  "error": {
      "code": 404,
      "message" : "Job not found"
  }
}
```
Using proper error codes pools may properly inform miners of the condition of their solution. If solution is accepted without errors it means is not stale. Any other condition, including staleness, could be easily derived from this very simple coding like for Http
- Errors 2xx not really an error but stale condition
- Errors 3xx lack of authorization (the worker is not authorized)
- Errors 4xx data error (either job not found or bad solution)
- Errors 5xx server error (internal server processng error)
