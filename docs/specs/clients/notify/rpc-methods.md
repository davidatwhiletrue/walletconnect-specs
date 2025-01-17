# RPC Methods

Methods can be called after the client [registers an Identity Key](../../clients/core/identity/identity-keys.md#key-registration).

All methods contain an "auth" field in the `params` (for request) or `result` (for response) which is a signed JWT. JWTs are specified in the [authentication](./authentication.md) document.

All methods follow the [JSON-RPC 2.0 spec](https://www.jsonrpc.org/specification).

## Getting & watching subscriptions

```mermaid
sequenceDiagram
  autonumber
  participant C as Client
  participant N as Notify Server

  activate C
  C->>+N: Get did.json
  N-->>-C: did.json
  Note over C: Derive req topic from did.json
  Note over C: Generate persistant key kY
  Note over C: Generate res topic from persistant key kY
  Note over C: Subscribe to res topic
  loop
	  C->>+N: REQ wc_notifyWatchSubscriptions({}) on req topic
	  N-->>-C: RES wc_notifyWatchSubscriptions(subs) on res topic
    loop
		  Note over N: on subscriptions updated
		  activate N
		  N->>C: REQ wc_notifySubscriptionsChanged(subs) on res topic
		  C--xN: RES wc_notifySubscriptionsChanged({}) on res topic
		  deactivate N
      Note over N: 1 day after step 3 no more updates
    end
    Note over C: 1 day after step 3 gobackto step 3
    Note over C: After going offline gobackto step 3
  end
  deactivate C
```

Notes:
- Each client will call `wc_notifyWatchSubscriptions` to receive subscription updates for a blockchain account
- `subs` contains the full subscription state, not delta or transactional differences
- `wc_notifyWatchSubscriptions` response and `wc_notifySubscriptionsChanged` request messages expires after 5 minutes to reduce mailbox load and these updates are ephemeral anyway
  - If client goes offline for 5 minutes it must call `wc_notifyWatchSubscriptions` again to continue receiving updates
- Notify server has an "watcher" timeout and will no longer send updates after 1 day of not calling `wc_notifyWatchSubscriptions`
  - Client must call `wc_notifyWatchSubscriptions` at least once a day to continue receiving updates

### wc_notifyWatchSubscriptions

Watches for updates to subscriptions for an account. Each update will result in a `wc_notifySubscriptionsChanged` request to the client. Calling this will also immediately trigger a `wc_notifySubscriptionsChanged` with the current subscription state.

Calling this method will create a "watcher" on the Notify Server for the given account and `iss`. A symkey and topic is derived from `kY` (which is specified under Response below) and saved for future updates. This derived topic is used to send the method response, and future updates to an account's subscriptions are also sent. Updates will stop and the watcher removed after the watcher timeout, which is 1 day.

If this method is called again with the same `iss`, it will update the watcher with the new account, reset the watcher timeout, and re-derive the symkey/topic from the current `kY` and Notify Keys. Clients should re-use the same `iss` and `kY` for the same account on the same device to avoid abandoned topics and improve performance. To watch multiple accounts at the same time, separate `iss` and `kY` must be used for each account otherwise it will either update the account being watched (if `iss` is the same), or result in receiving updates for multiple accounts on the same topic (if `kY` is the same).

**Request**

```jsonc
{
  "watchSubscriptionsAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 300      |
| Tag     | 4010     |
```

Topic: hash of key agreement public key from [Notify Server Authentication](./notify-server-authentication.md).

Message uses type 1 envelope with the client's persistant private key.

**Response**

```jsonc
{
  "responseAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 300      |
| Tag     | 4011     |
```

Topic: hash of symmetric key derivation of client's persistant private key and key agreement public key from [Notify Server Authentication](./notify-server-authentication.md).

### wc_notifySubscriptionsChanged

Used to indicate a change to subscriptions has occurred.

**Request**

```jsonc
{
  "subscriptionsChangedAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 300      |
| Tag     | 4012     |
```

Topic: same as wc_notifyWatchSubscriptions response.

**Response**

```jsonc
{
  "responseAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 300      |
| Tag     | 4013     |
```

Topic: same as wc_notifyWatchSubscriptions response.

## wc_notifySubscribe

Used to subscribe notify subscription to a peer through subscribe topic. Response is expected on the response topic.

**Request**

```jsonc
{
  "subscriptionAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 300      |
| Tag     | 4000     |
```

Topic: hash of key agreement public key from [Notify Server Authentication](./notify-server-authentication.md).

Message uses type 1 envelope with the client's persistant private key.

**Response**

```jsonc
{
  "responseAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 2592000  |
| Tag     | 4001     |
```

Topic: hash of symmetric key derivation of client's persistant private key and key agreement public key from [Notify Server Authentication](./notify-server-authentication.md).

```mermaid
sequenceDiagram
  autonumber
  participant U AS Wallet
  participant W AS SDK
  participant R as Relay
  participant N AS Notify
  participant D as Notify DB

  activate U
  U->>+W: User opens app
  Note over W: Prepare request
  Note over W: notify_subscribe = signJwt(params, identityKey)
  W->>+N: notify_subscribe + publicKeyY ON reqTopic
  Note over N: Generate P
  N->>+D: Store P @ domain,account
  D-->>-N: ACK
  Note over N: response = signJwt(identityKey)
  N->>-W: response ON resTopic
  W->>-U: UI updates
  deactivate U
```

## Detailed request & response pattern

This is used by `wc_notifyWatchSubscriptions` and `wc_notifySubscribe`

```mermaid
sequenceDiagram
  autonumber
  participant U AS Wallet
  participant W AS SDK
  participant I AS Identity Server
  participant R as Relay
  participant N AS Notify
  participant D as Notify DB
  participant A as dApp

  %% Request
  activate U
  U->>+W: User subscribes to dapp
  W->>+A: Wallet fetches publicKeyX from the did:web document
  A->>-W: publicKeyX
  Note over W: Generate publicKeyY
  Note over W: S = deriveSymmetric(privateKeyY, publicKeyX)
  Note over W: resTopic = sha256(S)
  W->>+R: subscribe(resTopic)
  R-->>-W: ACK
  Note over W: reqTopic = sha256(publicKeyX)
  Note over W: notify_subscribe = signJwt(params, identityKey)
  W->>R: notify_subscribe + publicKeyY ON reqTopic
  activate R
  R->>N: fwd
  activate N
  R-->>W: ACK (ignored)
  N-->>R: ACK
  deactivate R
  N->>+D: get privateKeyX
  D-->>-N: ACK
  Note over N: S = deriveSymmetric(privateKeyX, publicKeyY)
  Note over N: resTopic = sha256(S)
  Note over N: req = decrypt(S, msg)
  N->>+I: Get SIWE for identity key
  I-->>-N: SIWE CACAO
  Note over N: authenticate(siwe, req) -- error if unauthorized
  Note over N: Handle request ...
  N->>+D: Read/write to DB
  D-->>-N: ACK
  Note over N: response = signJwt(..., identityKey)
  N->>+R: response ON resTopic
  activate R
  R->>W: fwd
  R-->>N: ACK
  deactivate N
  W-->>R: ACK
  deactivate R
  W->>-U: UI updates
  deactivate U
```

## wc_notifyMessage

Used to publish a notification message to a peer through notify topic. Response is expected on the same topic.

**Request**

```jsonc
{
  "messageAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 2592000  |
| Tag     | 4002     |
```

Topic: notify topic.

**Response**

```jsonc
{
  "responseAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 2592000  |
| Tag     | 4003     |
```

Topic: notify topic.

## wc_notifyUpdate

Used to update a notify subscription with a new notify subscription, replacing an existing notify subscription through notify topic.

**Note:** this method is atomically performing two methods (wc_notifyDelete + wc_notifySubscribe)

**Request**

```jsonc
{
  "updateAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 300      |
| Tag     | 4008     |
```

Topic: notify topic.

**Response**

```jsonc
{
  "responseAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 2592000  |
| Tag     | 4009     |
```

Topic: notify topic.

## wc_notifyDelete

Used to inform the peer to close and delete a notify subscription through notify topic. The reason field should be a human-readable message defined by the SDK consumer to be shown on the peer's side.

**Request**

```jsonc
{
  "deleteAuth": string 
}

| IRN     |          |
| ------- | -------- |
| TTL     | 2592000  |
| Tag     | 4004     |
```

Topic: notify topic.

**Response**

```jsonc
{
  "responseAuth": string
}

| IRN     |          |
| ------- | -------- |
| TTL     | 2592000  |
| Tag     | 4005     |
```

Topic: notify topic.
