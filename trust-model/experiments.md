# Trusted server experiments for the FLEDGE Origin Trial

## Summary

One part of the [FLEDGE proposal](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) that Google Ads is exploring is what technologies can be used to add trust guarantees to servers.

We’ve written up an [overview of various theoretical techniques here](https://github.com/google/ads-privacy/blob/master/trust-model/trust_techniques.md) and now in this document we’re listing out some of the possible experiments we could run to see how those technologies perform.

This document is not a commitment of work and may be changed in the future.  We intend to publish the results of these experiments as we find them.

Note that many of these experiments require changes to the FLEDGE API.  We plan on experimenting with these API changes by making use of the public [FLEDGE Shim](https://github.com/google/fledge-shim) - this will let us test out changes without needing to build browser support for them.  We hope to run some of these experiments before a FLEDGE Origin Trial.

## Experiment goals

1. Explore trust techniques.  The aim is to run a broad variety of experiments, not necessarily to go deep into any one technique to start with.
1. Add to the industry discussion about trust models by measuring:
   1. Technology overhead, in both client CPU, server CPU, latency, and network used.
   1. Technology practicality, in terms of what scale these technologies can go up to currently.
   1. Where the tradeoffs are between client vs server compute.

### Non-goals

1. It’s not a goal to advocate for any one specific technology here.

## Experiment descriptions

These are the experiments that we would like to run in the next few quarters, when they are ready.

* Metrics we plan to capture include:
* Request latency and throughput.
* Server resource usage.
* Browser resource usage, as compared to the FLEDGE API.
* Network bandwidth used.

#### Baseline remarketing key/value cache

As part of the FLEDGE proposal, both [buyers and sellers](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#31-fetching-real-time-data-from-a-trusted-server) have the ability to specify trusted servers that can supply additional signals.  These two servers are both effectively key/value caches so our first step is to build such a server for a baseline.

* Needs extensions to the FLEDGE API? **No**
* Technology under test? **None, basic setup only.**

### Secure enclaves

#### Trusted Execution Environment overhead benchmarking

Extend the key/value cache to use [Trusted Execution Environments (TEEs)](https://github.com/google/ads-privacy/tree/master/trust-model/tee) and measure what the additional overhead is.

TEEs are designed to wrap around existing code and provide protections and remote attestation.

* Needs extensions to the FLEDGE API? **No**
* Technology under test? **TEEs**

#### Trusted Server Worklets

In the proposal [linked here](https://github.com/WICG/turtledove/issues/154), we suggested moving some of the FLEDGE JS processing to be run in a trusted server instead.

There are major questions here about whether the overhead of building such a server is worthwhile, which is what we want to test out.

* Needs extensions to the FLEDGE API? **Yes**
* Technology under test? **TEE+Sandbox**


#### Trusted Worklet and Auction

Beyond running JS worklet code in a trusted environment, can we run a full auction there too?  This is more speculative than the experiment above and is not yet designed.

* Needs extensions to the FLEDGE API? **Yes**
* Technology under test? **TEE+Sandbox**

### Secure Multi-Party Computation (MPC)

#### MPC key/value server

Extend the key/value cache to use MPC and measure what the additional overhead is.

Unlike the similar TEE experiment above, this would be a new implementation.

* Needs extensions to the FLEDGE API? **Yes**
* Technology under test? **MPC**

#### Dovekey Auction

[As written up here](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md) and [here](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md), Dovekey Auction proposes moving an ad auction to run inside a MPC cluster.

We would like to build out this cluster and benchmark the MPC overhead to see how practical this system is.

* Needs extensions to the FLEDGE API? **Yes**
* Technology under test? **MPC**

#### Fully featured Dovekey Auction

The Dovekey Auction doc covered not only an auction but also a number of additional common ads features, e.g. frequency capping and mute-this-ad.  An extension of the experiment above is to add those features to the auction.

* Needs extensions to the FLEDGE API? **Yes**
* Technology under test? **MPC**
