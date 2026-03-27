---
layout: post
title:  "Building empires on blockchain"
date:   2025-02-21 14:12:06 +0000
categories: blockchain
---
## Overview

Cryptocurrencies have become mainstream news with early entires like Bitcoin
soaring in value as speculators rush in to reap the rewards of its popularity.
But for most people, the machinery that makes all this work is hidden from
view in a sea of social media. In this article, I hope to disentangle the
meme coin from the blockchain and show that behind all the media hysteria there
lies a highly utititarian system that can be a huge benefit to mankind.

## Bitcoin

You would have to lead a very sheltered life not to have heard of Bitcoin.
According to Wikipedia, "Bitcoin was invented in 2008 by Satoshi Nakamoto,
an unknown entity (person or persons)". But in practice it is just another
project hosted on Github like millions of others.

Today, go into any gas station in the U.S. and there are vending machines
for bitcoin which in a country where other forms of gambling are restricted
makes it the exception.

Bitcoin was originally run on ethusiast's PCs. You could download and build
the code and run it in the background to create "value" in the form of bitcoins.
But very soon big business became involved and PCs became expensive GPUs
and the GPUs became custom ASICs until it was no longer viable for individuals
to mine bitcoin. Now bitcoin is mined in vast warehouses in Kazakstan and
other places with cheap energy. Bitcoin uses more energy than many medium
sized nations.

The only real use of Bitcoin is to own Bitcoin, making it a bit like a popular
dance tune naming itself. Attempts have been made to add extra services to
bitcoin such as BRC-20, but other networks, especially Ethereum do this a lot
better and cheaper.

Bitcoin uses a network of computers, a bit like Bittorrent or Napster, to
create chains of blocks of transactions. Each transaction on bitcoin
represents the transfer of bitcoin from one account to one or more other
accounts. The miners, who have to do hard sums to win a competition,
get to pick which transactions are included in a block and are paid
a reward - in bitcoin - for doing so.

## Ethereum

The Ethereum chain was an offshoot of Bitcoin devised by bitcoin magazine
editor Vitaly Dmitrievich Buterin (Vitalik). Vitalik wanted bitcoin
to be more useful than just a store of "wealth". He worked with other
computer scientists like Gavin Wood to add a virtual machine to Ethereum
known as the EVM.

The EVM enables Ethereum to be used like a stock exchange, trading not just
one token, ETH, but many others besides. There are currently hundreds of thousands
of tokens traded on Ethereum.

The EVM has become the dominant force in blockchain trading with $60bn of tokens
and $122bn in stablecoins and $182bn in "bridged tokens" currently available on the platform.
Ethereum collects $800M in fees per day at the time of writing and trades $1.9bn per day.
This compares to the London Stock Exchange which has $3.4tn despite Brexit.
So blockchains are still behind traditional markets, but much could change in the
next decade.

See https://defillama.com/chain/Ethereum

Ethereum now uses a "proof of stake" consensus mechanism which means that you
do not need the vast warehouses in Kazakstan to power it and it is still possible
to run an Ethereum node on an ordinary server.

## Other chains

There are many other blockchains of which this is an abbrevaited list:

* Aptos
* SUI
* Solana
* Fuel
* Nano
* Zcash
* Polygon
* Arbitrum
* Optimism
* Cardano
* Avalanche

See https://en.wikipedia.org/wiki/List_of_blockchains for a more complete list


## How trading works on a blockchain.

If you want to buy some token using Euros, for example, you must first convert
your Euros into a "stablecoin" such as EURC using an "on ramp" which
is usually a centralised service. You pay circle.com via Swift and circle
then mints you some of their token in return.

https://www.circle.com/eurc

Likewise, if you want to convert EURC to Euros, you contact circle.com
who will reverse the process.

Stablecoins, such as EURC and USDC track the value of "fiat" currencies
such as the Euro or USD and are often the first port of call in the
blockchain experience.

It is easy to do this through web servcies like "Metamask" which also handle
your "wallet" or private key that enables you to send transactions to the network.

EURC is a smart contract that stores your balance in secure storage or "state".
you can see the source code of this contract here:

https://etherscan.io/token/0x1abaea1f7c830bd89acc67ec4af516284b1bc33c#code

You can use this contract to transfer EURC from yourself to other users
on Ethereum.

## Swaps

On Ethereum, there is a standard called "ERC20" which enables tokens of different
kinds to be exchanged.

Special contracts called Automatic Market Makers or AMMs for short allow you to swap
tokens of one kind for another. So for example, if you wanted to swap your EURC
for USDT, a dollar tracking stablecoin, you can do this using an AMM.

The top AMM contracts on ethereum are:

* Uniswap V2
* Uniswap V3
* Curve

But there are many, many others. Each provide a different fee structure.

AMMs work by getting investors to put value into the market in the form of
a pair of tokens. This creates "liquidity" in the market and allows
other users to swap between two or more different tokens.

Uniswap uses a hyperbolic curve to determine price depending on the
reserves of tokens on either side. Curve uses a modified hyperbola
and is used primarily for stablecoin trading.

As the amount of token on one side of the trade changes, the price
of the swap is determined by the curve. If a reserve becomes depleted
its price will rise, incentivising investors to put more tokens
on that side.

These contracts have come to dominate with Uniswap holding roughly one third
of the market on Ethereum.

The authors of these contracts have become billionaires leading to the Curve
Finance CEO buying two mansions in Melbourne.

https://www.theblock.co/post/232464/curve-finance-ceo-michael-egorov-wife-mansions-australia

