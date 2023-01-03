---
simd: '0016'
title: Application Fees (Write-lock fees)
authors:
  - Godmode Galactus (Mango Markets)
category: Fees
type: Fees
status: Draft
created: 2022-12-23
---

## Problem

According to the discussion on the following issue:
https://github.com/solana-labs/solana/issues/21883

During network congestion, many transactions in the network cannot be processed because they all want to write lock similar accounts. When a write lock on an account is taken by a transaction batch no other batch can use this account in parallel, so only transactions belonging to a single batch are processed correctly and all others fail. Then the failed transactions are forwarded to the leader of the next slot. There are multiple accounts of OpenBook (formerly serum), mango markets, etc which are used in high-frequency trading and by market makers. During the event of extreme congestion these clients are incentivized to send as many transactions as possible to the network so that some of their transactions may pass through, this adds more congestion to the network.

## Solution

As a high-performance cluster, we want to incentivize players which feed the network with correctly formed transactions instead of spamming the network. Introduction of application fees (write account fees) would be interesting way to penalize the bad actors and applications can rebate these fees to the good actors. This means the application has to decide who to rebate and who to penalize.

### Application fees in working

As an owner of a writable account that is used a lot in the network, a program can use the application fees program to assign it an application fee. This application fee is applied to every transaction which will try to lock the account in writable mode. This means even if the transaction eventually fails the application fee will be charged to the payer.

These application fees will be tracked by the bank as it is running transaction batches and eventually when the bank is frozen it will dispatch all the application fees that were collected to the respective writable accounts. Then the owners of the writable accounts can collect these fees directly from the writable accounts.

Programs/Owners can also invoke or cpi instructions like Rebate or RebateAll to effectively cancel this application fee for good actors. So in the end application fee won't be charged to these actors. The rebate will be effective only if the transaction is executed and it wont be effective if the transaction fails.

### Application fee program

A new native solana program with id `App1icationFees1111111111111111111111111111` will handle updating and rebating the application fees. It will have following instructions : 

#### UpdateFees
This instruction will update application fees for a writable account. Internally it will create a pda for a writable account (if the pda does not exists) and update with the fees information. If the fees is set to 0 the program will deallocate the pda.
It requires : 
* Owner of the writable account (signer)
* Writable account (writable)
* Derived address calulated from writable account for application fees program (writable)
* Payer (writable) (signer)
* System program


#### Rebate
This instruction will remove fees for a writable account in a transaction.
It requires : 
* Owner of the writable account (signer)
* Writable account (writable)

#### Rebate all
This instruction will remove all the fees from all the writable accounts belonging to a owner.
It requires :
* Owner (signer)


## POC
A draft proof of concept has been implemented. \

[Repo](https://github.com/blockworks-foundation/solana.git) \
Branch : `application-fees` \
[Pull Request](https://github.com/blockworks-foundation/solana/pull/18)

## Example contract

Here is an working example to test application fees with a smart contract (work in progress).
[Example](git@github.com:godmodegalactus/paper-clip-maximizer.git)

## Mango V4 Usecase
With this feature implemented Mango-V4 will be able to charge users who spam risk-free aribitrage or spam liquidations by increasing application fees on perp-markets, token banks and mango-user accounts.
#### Perp markets
Application fees on perp liquidations, perp place order, perp cancel, perp consume, perp settle fees.
Rebates on : successful liquidations, consume events, HFT marketmaking refresh (cancel all, N* place POST, no IOC).

#### Token vaults
Application fees on openorderbook liquidations, deposit, withdrawals.
Rebate on successful liquidations, place IOC & fill in isolation, HFT marketmaking refresh (cancel all, N* place POST, no IOC).

#### Mango accounts 
Application fees on all liquidity transactions, consume events, settle pnl, all user signed transactions.
Rebate on transaction signed by owner or delegate, successful liquidations, settlements, consume events.