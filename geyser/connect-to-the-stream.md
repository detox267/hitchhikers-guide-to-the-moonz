# 📡 Connect to the Stream

You know where the stream lives.

Now we open the pipe.

The Moonz Geyser uses standard gRPC over TLS and HTTP/2.

No Moonz account.

No API key.

No access token.

Just a gRPC client and the public Moonz protocol files.

{% hint style="success" %}
**Public endpoint**

`grpc.moonz.fun:443`

Authentication is not required.
{% endhint %}

## 🌊 What You Need

The public developer toolkit currently uses:

```text
@grpc/grpc-js 1.14.4

@grpc/proto-loader 0.8.1

protobufjs 7.6.5
```

It also includes the protocol definitions required by the client:

```text
proto/geyser.proto

proto/solana-storage.proto
```

The current public toolkit version is:

```text
0.7.2
```

## 🚀 Fastest Way In

The public toolkit already contains a working Node example.

Inside:

```text
examples/node
```

install its dependencies:

```bash
npm ci
```

Create the local environment file from the supplied example:

```bash
cp ../../.env.example .env
```

The environment contains:

```text
MOONZ_GEYSER_ENDPOINT=grpc.moonz.fun:443
MOONZ_PROGRAM_ID=DBc9SEQghiJUj52YPqTKk8R4CMRgagBxi2LU1yBbeMpk
```

There is no authentication secret to add.

## 🧪 Test the Connection First

Before opening a live subscription, check the endpoint.

```bash
node client.js --check
```

The example calls:

```text
Geyser.GetVersion
```

If the connection succeeds, you should see the endpoint, Moonz Program ID, server version and:

```text
Connection check: PASS
```

The client then closes without opening a subscription.

That makes `--check` a useful first diagnostic.

## 🔐 TLS Connection

The client connects using:

```js
const client =
  new Geyser(
    ENDPOINT,
    grpc.credentials.createSsl()
  );
```

The important part is:

```js
grpc.credentials.createSsl()
```

The public endpoint uses TLS.

Do not create an insecure connection to:

```text
grpc.moonz.fun:443
```

## 📦 Load the Yellowstone Protocol

The example loads:

```text
geyser.proto
```

with `@grpc/proto-loader`.

The current public example uses these options:

```js
const definition =
  protoLoader.loadSync(
    path.join(
      protoDir,
      "geyser.proto"
    ),
    {
      keepCase: true,
      longs: String,
      enums: String,
      defaults: false,
      oneofs: true,
      includeDirs: [
        protoDir,
        protobufRoot
      ]
    }
  );
```

Then:

```js
const loaded =
  grpc.loadPackageDefinition(
    definition
  );

const Geyser =
  loaded.geyser.Geyser;
```

This gives the application the Yellowstone compatible `Geyser` client.

## 🔢 Why longs Are Strings

Notice:

```js
longs: String
```

Blockchain values can exceed the integer range JavaScript can safely represent as a normal `number`.

Loading protobuf integer values as strings helps preserve their exact value.

That is particularly important for indexing infrastructure.

Do not casually convert large raw values into JavaScript numbers.

## 🧠 Why keepCase Is Enabled

The example also uses:

```js
keepCase: true
```

That means protobuf field names remain in their original form.

For example:

```text
account_include

account_exclude

account_required

write_version
```

rather than being automatically transformed into another JavaScript naming convention.

That makes the JavaScript structures line up closely with the protocol definitions.

## 💓 Keep the Connection Healthy

The public example configures gRPC keepalive settings:

```js
{
  "grpc.keepalive_time_ms": 60000,
  "grpc.keepalive_timeout_ms": 20000,
  "grpc.keepalive_permit_without_calls": 1
}
```

These settings help maintain the HTTP/2 connection.

The Moonz service also sends its own stream heartbeat approximately every fifteen seconds.

We will deal with reconnect logic and production health later.

For now, the important thing is that the connection is designed to stay open.

## 📞 Check the Server Version

The example exposes a small helper around:

```text
Geyser.GetVersion
```

Conceptually:

```js
client.GetVersion(
  {},
  metadata,
  callback
);
```

The current client creates empty gRPC metadata:

```js
function metadata() {
  return new grpc.Metadata();
}
```

There is no `x-token`.

There is no Authorization header.

The endpoint is public.

## 🌊 Open the Subscription

Once the connection check has passed, start the live stream with:

```bash
node client.js
```

The client opens:

```js
const stream =
  client.Subscribe(
    metadata()
  );
```

This is a bidirectional gRPC stream.

The client writes its requested subscription scope.

The server sends matching Moonz updates back through the same stream.

## 🎯 Subscribe to Moonz Transactions

The current public example requests Moonz transactions with:

```js
transactions: {
  moonz_transactions: {
    vote: false,
    failed: false,

    account_include: [
      PROGRAM_ID
    ],

    account_exclude: [],
    account_required: []
  }
}
```

The crucial field is:

```text
account_include
```

containing the Moonz Program ID.

The server enforces this scope.

You cannot replace it with an unrelated program and turn the service into a general Solana transaction stream.

## 🌙 Subscribe to Moonz Program Accounts

The same request can subscribe to accounts directly owned by the Moonz program:

```js
accounts: {
  moonz_accounts: {
    owner: [
      PROGRAM_ID
    ],

    account: [],
    filters: []
  }
}
```

Again, the Moonz Program ID defines the permitted scope.

This stream can deliver program owned accounts such as Moonz state accounts.

## ⚠️ This Does Not Include Every Vault

Remember the distinction from the SDK section.

A Moonz Launch State account is owned by the Moonz program.

An SPL Token vault is owned by the Solana Token Program.

So:

```text
Moonz program account
        ↓
Included by Moonz owner subscription

SPL Token vault
        ↓
Not automatically included
```

Do not assume an account belongs in the stream simply because it participates in Moonz.

Account ownership determines the owner scoped account stream.

## ✅ Commitment

The request specifies:

```js
commitment:
  "CONFIRMED"
```

That is currently the only accepted commitment level.

Do not request:

```text
PROCESSED

FINALIZED
```

from the Moonz public bridge.

The service currently exposes confirmed Moonz activity.

## 📡 The Complete Basic Request

The working request looks like this:

```js
stream.write({
  transactions: {
    moonz_transactions: {
      vote: false,
      failed: false,

      account_include: [
        PROGRAM_ID
      ],

      account_exclude: [],
      account_required: []
    }
  },

  accounts: {
    moonz_accounts: {
      owner: [
        PROGRAM_ID
      ],

      account: [],
      filters: []
    }
  },

  commitment:
    "CONFIRMED"
});
```

That single request asks for both supported live data surfaces.

Moonz scoped transactions.

And Moonz program owned accounts.

## 🧱 Server Scope Is Enforced

The public endpoint does not simply trust clients to behave.

Moonz restricts subscription scope on the server.

For transactions, the public interface does not support broadening features such as:

```text
Unrelated program subscriptions

Vote transactions

Failed transactions

Signature filtering

Required account filtering

Account exclusions
```

For account subscriptions, the public interface does not currently support:

```text
Individual account address filters

Account data filters

Account data slices

Unrelated account owners
```

The intended interface is deliberately narrow.

**Give me the Moonz program stream.**

## 🚫 PERMISSION_DENIED

If the requested subscription tries to exceed the permitted Moonz scope, the server can return:

```text
PERMISSION_DENIED
```

That normally means the subscription request itself is outside the public Moonz boundary.

Do not solve that by repeatedly reconnecting with the same invalid request.

Fix the subscription.

## 🚦 RESOURCE_EXHAUSTED

The service can also return:

```text
RESOURCE_EXHAUSTED
```

The current public limits are:

```text
5 simultaneous streams
per real client IP

100 total service clients

10 second subscription
handshake window

4 MiB maximum
gRPC message size
```

A production integration should treat capacity errors differently from ordinary connection drops.

We will cover retry behaviour in the production reliability page.

## 🧩 UNIMPLEMENTED

The public Moonz bridge currently supports:

```text
Subscribe
Ping
GetVersion
```

Other Yellowstone RPCs may return:

```text
UNIMPLEMENTED
```

Do not assume the presence of the full Yellowstone protobuf means every Yellowstone RPC is implemented.

The protocol definitions define the language.

The Moonz integration specification defines the supported surface.

## 💓 Ping the Stream

The example also writes:

```js
stream.write({
  ping: {
    id: 1
  }
});
```

The corresponding response can arrive as:

```js
update.pong
```

with the same ping identifier.

The server can also independently send heartbeat messages through:

```js
update.ping
```

approximately every fifteen seconds.

## 📥 Receive Updates

The Node client listens with:

```js
stream.on(
  "data",
  update => {
    console.log(update);
  }
);
```

But the useful approach is to inspect which update type arrived.

```js
if (update?.transaction) {
  // Moonz transaction
}

if (update?.account) {
  // Moonz program account
}

if (update?.ping) {
  // Server heartbeat
}

if (update?.pong) {
  // Response to client ping
}
```

That becomes the entry point to your ingestion pipeline.

## ⚡ Transaction Updates

A transaction update contains information such as:

```text
slot

filters

transaction

signature
```

The public example prints the transaction signature as base64 simply to demonstrate that bytes arrived.

A real indexer will go further.

It will decode the transaction.

Inspect the Moonz instructions.

Read logs.

Decode events.

And store the result.

That comes later in this section.

## 🌙 Account Updates

An account update contains information such as:

```text
slot

filters

pubkey

owner

data

write_version
```

The public example prints the account public key and owner as base64.

Again, that is only a connectivity example.

A real Moonz integration will decode the account bytes using the Moonz IDL and identify the account type.

## ⚠️ Remember write_version

The Moonz bridge currently reports:

```text
write_version = 0
```

because the upstream account source does not expose validator native write version semantics.

Do not use this field for native validator write ordering.

We will build safer ordering and deduplication rules later.

## 🛑 Shut Down Cleanly

A streaming application should close both the subscription and the client.

The public example does:

```js
const shutdown = () => {
  try {
    stream.cancel();
  } catch (_) {}

  try {
    client.close();
  } catch (_) {}

  process.exit(0);
};
```

and connects it to:

```js
process.on(
  "SIGINT",
  shutdown
);

process.on(
  "SIGTERM",
  shutdown
);
```

So pressing:

```text
Ctrl+C
```

does not simply abandon the stream.

It cancels the subscription and closes the gRPC client.

## 🧪 Your First Connection

The complete beginner flow is:

```text
Install dependencies
        ↓
Create .env
        ↓
node client.js --check
        ↓
GetVersion succeeds
        ↓
node client.js
        ↓
Subscribe
        ↓
Moonz updates arrive
```

Nothing has been indexed yet.

Nothing has been decoded yet.

But the pipe is open.

{% hint style="success" %}
**Connection established.**

You now have a live confirmed stream from the Moonz program into your application.

Next we make the stream useful.
{% endhint %}

## 🎯 Next Stop

We can connect.

Now we decide exactly what Moonz data we want to consume.

Next stop:

**Subscribe to Moonz**
