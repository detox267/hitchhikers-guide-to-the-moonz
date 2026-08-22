# 🌊 Meet the Moonz Geyser

The Moonz SDK gives you a telescope.

You point it at Solana and ask questions.

What tokens exist?

What phase is this launch in?

What is the current market?

What happened in this transaction?

That works extremely well when your application wants information on demand.

But eventually some developers need something different.

They do not want to keep asking.

They want the data to arrive.

Welcome to the Moonz Geyser.

{% hint style="info" %}
**The Moonz Geyser is the live data surface for developers building continuous Moonz infrastructure.**

It provides a public Yellowstone compatible gRPC stream scoped specifically to the Moonz program.
{% endhint %}

## 🌊 From Telescope to Stream

With the SDK, your application might do this:

```text
Application
    ↓
Ask Solana
    ↓
Receive Moonz state
```

With Geyser, the direction changes.

```text
Moonz activity
    ↓
Moonz Geyser
    ↓
Continuous gRPC stream
    ↓
Your application
```

Instead of repeatedly checking whether something changed, your infrastructure can stay connected and process Moonz activity as it arrives.

## 🛰️ The Public Endpoint

Moonz provides a public mainnet gRPC endpoint:

```text
grpc.moonz.fun:443
```

Network:

```text
Solana Mainnet
```

Transport:

```text
TLS
HTTP/2
gRPC
```

Service:

```text
geyser.Geyser
```

Commitment:

```text
CONFIRMED
```

Current service version:

```text
moonz-program-geyser/0.7.0
```

## 🔓 No API Key

The public Moonz gRPC endpoint does not require:

```text
API keys
Access tokens
Moonz authentication credentials
```

A developer can connect directly to the public endpoint.

{% hint style="success" %}
**No Moonz account is required to consume the public stream.**

The stream is intended to be an open integration surface for developers and data platforms.
{% endhint %}

## 🎯 Moonz Scoped

There is an important boundary.

This is not a general Solana data service.

The Moonz Geyser is deliberately scoped to the Moonz program.

The Moonz Program ID is:

```text
DBc9SEQghiJUj52YPqTKk8R4CMRgagBxi2LU1yBbeMpk
```

Transaction subscriptions are restricted to confirmed transactions involving that program.

Account subscriptions are restricted to accounts directly owned by the Moonz program.

So:

```text
Solana Mainnet
      ↓
Moonz Program activity
      ↓
Moonz Geyser
      ↓
Your integration
```

You cannot turn the Moonz endpoint into a general Solana firehose.

That is intentional.

## 📡 What Can You Subscribe To?

The current service supports:

```text
Geyser.Subscribe
Geyser.Ping
Geyser.GetVersion
```

`Subscribe` is where the interesting work happens.

It can deliver Moonz scoped transaction activity and Moonz program owned account updates.

We will build those subscriptions in the next pages.

## 🧠 Why Geyser Exists When We Already Have watch()

You may be thinking:

**Did we not just build live listeners with the SDK?**

Yes.

And for many applications, that is exactly what you should use.

The SDK watcher:

```ts
moonz.watch()
```

is excellent for:

```text
Bots
Applications
Notifications
Live interfaces
Simple trade feeds
Token watchers
```

Geyser is aimed at another level of integration.

Think:

```text
Continuous ingestion
Persistent indexers
Analytics infrastructure
Large event pipelines
Database synchronisation
Protocol monitoring
Data platform integrations
```

The two tools solve related but different problems.

## 🔭 SDK or Geyser?

A useful rule is:

```text
Need to read Moonz?
Use the SDK.

Need a convenient live listener?
Use the SDK watcher.

Need continuous infrastructure ingestion?
Use Geyser.
```

You do not need Geyser just because it exists.

Use it when your architecture actually benefits from a persistent stream.

## 🧬 Yellowstone Compatible

The Moonz service speaks the Yellowstone gRPC protocol.

The public developer toolkit includes the validated protocol definitions:

```text
proto/geyser.proto
proto/solana-storage.proto
```

That means developers familiar with Yellowstone can work with a familiar gRPC message structure.

But there is an important distinction.

## ⚠️ Not a Validator Native Plugin

The Moonz endpoint is Yellowstone compatible.

It is **not** a validator native Geyser plugin.

That matters because developers should not assume that every field or behaviour offered by a native Yellowstone provider exists here.

The Moonz service exposes the parts required for the Moonz program scoped stream.

{% hint style="warning" %}
**Yellowstone compatible does not mean unrestricted Yellowstone.**

Use the Moonz integration specification as the authority for what the endpoint actually supports.
{% endhint %}

## ✍️ Account write_version

One known compatibility difference is:

```text
write_version = 0
```

The current upstream account source does not expose validator native `write_version` information.

Moonz therefore reports the value as zero.

Do not use that field as though it represented native validator write ordering.

Later we will cover safer ways to process the stream.

## ✅ Confirmed Means Confirmed

The current Moonz Geyser supports:

```text
CONFIRMED
```

commitment.

That is the commitment level integrations should request.

The endpoint is not currently presented as a configurable processed, confirmed and finalized feed.

It is a confirmed Moonz stream.

## 💓 The Stream Has a Heartbeat

Accepted streams receive a Yellowstone compatible server heartbeat roughly every fifteen seconds.

That gives clients a way to know the connection is still alive even when Moonz activity is quiet.

A production consumer should treat connection health as part of the integration rather than assuming a silent socket will remain healthy forever.

We will cover that properly when we build the production reliability page.

## 📦 Maximum Message Size

The current public integration specification defines a maximum gRPC message size of:

```text
4 MiB
```

For normal Moonz program traffic this is primarily an integration boundary developers should be aware of when configuring their gRPC client.

## 🏗️ What Can You Build?

Once connected, the stream can become the front door to your own Moonz data system.

For example:

```text
Moonz Geyser
      ↓
Transaction processor
      ↓
Decode Moonz activity
      ↓
Database
      ↓
Your API
      ↓
Your product
```

That product could be:

* A Moonz explorer
* A live analytics platform
* A trade indexer
* A token discovery engine
* A charting backend
* A migration tracker
* A PCLS tracker
* A notification service
* A market data pipeline
* A third party integration

The Geyser does not decide what you build.

It gives you the stream.

## 🌙 Moonz Accounts and Token Accounts Are Different

There is one architectural detail we should establish early.

An account subscription can stream accounts directly owned by the Moonz program.

But SPL Token accounts are owned by the Solana Token Program.

That means token vault accounts are not automatically included merely because they belong to the architecture of a Moonz launch.

This distinction becomes important when building an indexer.

```text
Moonz Launch State
Owner
Moonz Program

Token Vault
Owner
SPL Token Program
```

We will handle these account relationships carefully when we build the subscription pages.

## 🧰 The Public Developer Toolkit

The public Moonz gRPC developer toolkit includes more than the protocol files.

It also contains:

```text
Verified Moonz Anchor IDL

Canonical events

Canonical instructions

Program account definitions

PDA seeds

Program metadata

Program verification information

Indexing guidance

Known limitations

Security limits

Working Node example
```

So an integration does not have to reverse engineer the Moonz program before it can begin consuming the stream.

## 🔐 Verified Program Source

The developer toolkit identifies the verified Moonz program source and the exact Anchor IDL used by the integration material.

That matters because a serious indexer should know which program interface it is decoding.

The stream tells you what happened.

The IDL tells you how to understand it.

## 🌊 The Big Picture

The complete flow starts to look like this:

```text
              SOLANA MAINNET
                    |
                    ↓
              MOONZ PROGRAM
                    |
          confirmed activity
                    |
                    ↓
             MOONZ GEYSER
                    |
             gRPC Subscribe
                    |
                    ↓
          YOUR DATA PIPELINE
             /            \
            ↓              ↓
      Transactions       Accounts
            |              |
            └──────┬───────┘
                   ↓
               Decoder
                   ↓
                Storage
                   ↓
            Your Application
```

This is where Moonz starts becoming infrastructure.

{% hint style="success" %}
**You already know how to read Moonz.**

Now you are going to learn how to keep a pipe connected to it.
{% endhint %}

## 📡 Next Stop

We know what the Moonz Geyser is.

Now we connect to it.

Next stop:

**Connect to the Stream**
