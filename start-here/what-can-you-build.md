# 🚀 What Can You Build?

Moonz gives you access to the protocol.

What you build on top of it is entirely up to you.

You might only need to recognise a Moonz token inside your application.

Or you might want to follow thousands of Moonz markets as they move through bonding, trading and liquidity changes.

There is no single way to build with Moonz.

So before we start digging through SDKs, accounts and program state, here are a few things you could create.

{% hint style="info" %}
**Think of these as destinations, not requirements.**

You do not need to build everything in this guide.

Pick the part of Moonz your application cares about and start there.
{% endhint %}

## 🔎 Moonz Token Detector

Start with one question.

**Is this a Moonz token?**

Give your application a token mint and determine whether that token belongs to the Moonz protocol.

It sounds simple because it is.

But that one answer can become the first step in a much larger integration.

### You could use it in

A token scanner

A wallet

A trading bot

A portfolio tracker

A blockchain explorer

An analytics platform

### The idea

Your application receives a mint address.

It checks the Moonz program.

It gets an answer.

**Moonz**

or

**Not Moonz**

Later in this guide we will build exactly that.

## 🌡️ Bonding Scanner

Moonz tokens begin their journey through bonding.

That journey is visible on chain.

So instead of simply showing that a token exists, you can build something that follows its progress.

Imagine opening a scanner and seeing:

**41% Bonded**

**76% Bonded**

**94% Bonded**

**99% Bonded**

You could watch markets approaching the next stage of their lifecycle and update your application as the underlying protocol state changes.

### You could build

A bonding leaderboard

A token discovery page

A bonding alert bot

A market terminal

A watchlist

A realtime progress display

The interesting part is that your application is not guessing where the token is.

It is reading Moonz.

## 🤖 Trading Bot

Maybe your application does not just want to watch Moonz.

Maybe it wants to trade.

The Moonz Trading SDK gives developers a way to integrate Moonz trading into their own applications.

A bot could recognise a Moonz token, understand its market and prepare the correct transaction for the user.

### You could build

A Telegram trading bot

A Discord trading bot

A custom trading terminal

A mobile trading interface

An automated strategy

A wallet integration

The user sees a Buy button.

Your application handles everything underneath.

We will get into that later.

For now, just remember that Moonz is not limited to trading through moonz.fun.

## 📊 Market Dashboard

Raw blockchain state is useful to developers.

It is not particularly friendly to humans.

That gives you something to build.

Take Moonz state and turn it into a market dashboard.

### You could display

Token lifecycle

Bonding progress

Current market state

Active quote asset

Price information

Market cap

Protocol reserves

Liquidity information

Creator information

A dashboard could focus on one Moonz token or the entire ecosystem.

You decide what matters.

## ⚡ Live Moonz Feed

Sometimes asking the chain what happened is not enough.

You want to know when it happens.

That is where realtime Moonz tooling becomes interesting.

Your application could listen for Moonz activity and react as the protocol moves.

### Imagine a feed showing

A new Moonz token launched

A buy happened

A sell happened

Bonding progressed

A token changed lifecycle state

A market entered a new state

PCLS activity occurred

From there you could turn the same data into alerts, analytics, trading signals or your own historical database.

Later in the guide we will meet the Moonz Geyser tooling built for this kind of work.

## 🛰️ Moonz Explorer

What if you want to go underneath the website completely?

Build an explorer.

Instead of showing users a generic Solana account, explain what that account actually means inside Moonz.

### An explorer could show

The token mint

The creator

The Moonz launch state

Protocol controlled vaults

Current lifecycle state

Market configuration

Quote asset

Relevant program accounts

This is where raw Solana starts turning into something humans can understand.

## 👑 Holder Intelligence

Here is a problem that matters more than it first appears.

Not every account holding tokens is a normal holder.

Moonz uses Program Derived Addresses and protocol controlled vaults as part of the protocol.

If an analytics platform does not understand that, a protocol vault can look like an enormous whale.

That can produce misleading holder information.

### A Moonz aware holder system can

Recognise protocol accounts

Separate protocol controlled supply from ordinary holders

Improve whale classifications

Produce clearer holder analytics

Give users better context

Sometimes the most useful integration is not adding more information.

It is understanding the information correctly.

## 🔄 PCLS Tracker

Moonz markets can evolve.

PCLS, the Program Controlled Liquidity Switch, allows the liquidity configuration of a Moonz market to change through the protocol.

That creates another opportunity for developers.

Your application can follow those changes rather than assuming a market will always remain exactly as it began.

### You could build

A PCLS activity tracker

A market state monitor

A quote asset tracker

A trading interface that reacts to market changes

An analytics system that records market transitions

A notification bot for PCLS activity

The market changes.

Your application notices.

## 🧠 Build Something We Did Not Think Of

This is the part we cannot document for you.

The SDKs give you tools.

The IDL gives you a language for understanding the program.

Solana gives you the chain.

Geyser gives you another way to listen.

What happens when somebody combines those pieces in a way we never expected?

That is the interesting part.

{% hint style="success" %}
**The goal of this guide is not to tell you what to build.**

It is to give you enough understanding of Moonz that you can decide for yourself.
{% endhint %}

## 🧭 Where Do We Go Next?

Before building any of these projects, we need to get you connected to Moonz.

Next stop:

**Don’t Panic**

We will install the Moonz SDK and make your first request to the protocol.
