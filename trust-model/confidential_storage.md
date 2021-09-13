# Confidential storage

## Summary

This explainer goes into some background about what confidential storage is, takes a high-level look at some techniques to build it, and then how we may want to make use of it to help make ads privacy safe.

### What is confidential storage?

Data stores, e.g. databases, are unaware of what keys are in the query (or cannot leak them) and the same with the values being retrieved.

## Confidential storage techniques

That sounds improbable at first glance, let’s see a few techniques of how this can be achieved.

### [Private information retrieval](https://en.wikipedia.org/wiki/Private_information_retrieval) (PIR)

The simplest way to ensure that a database doesn’t know what key a user has requested is for the user to request every single value in the database and then discard the ones that they don’t need.

That doesn’t work well in practice as it’s too inefficient.

There are two broad categories of alternatives which both have trade-offs:
1. Allow many copies of the database that do not communicate or collude.
1. Placing computational limits on what the single database is allowed to do.

The downside with all of these techniques is in the bandwidth and computation overhead necessary: they’re all varying degrees of worse than a standard database.

### Trusted Execution Environment (TEEs)

TEEs can provide ways to simulate PIR, for example it’s possible to set up a TEE so that the database is limited in what it can do with the user’s data: it can still see what’s being requested but it can’t do anything with that knowledge.

An example of this is included as part of [Project Oak](https://github.com/project-oak/oak/blob/0b63f58c4f716b7c7d87faad8fd480ee3296945a/examples/trusted_database/client/rust/src/main.rs#L73-L99).

The limitations here are those of TEEs in general: that they require hardware support and for the database server to be run inside a TEE setup.  The benefits are that the overhead is likely to be significantly less than PIR or MPC.

### Secure Multi-Party Computation (MPC)

MPC systems allow computation on top of encrypted data.  This can be implemented in a variety of ways.

In the [Dovekey Auction](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md) the MPC servers that are used to run the ad auction never see the unencrypted data.  Unless the MPC servers collude they cannot know anything about the ad auction or the user data.  The same applies to the MPC servers that train and evaluate machine learning models in the [SCAUP proposal](https://github.com/google/ads-privacy/tree/master/proposals/scaup).

The downsides here are that while MPC can theoretically run any program in a secure way, it often ends up being computationally too expensive to be practical for arbitrary use cases. However, for selected use cases, e.g. PIR based on secure 2-Party Computation, carefully crafted MPC protocols may be practical - we plan on experimenting with these in the future.

### Oblivious RAM

This is [a technique](https://en.wikipedia.org/wiki/Oblivious_RAM) which alters the memory access pattern of a computer program so that an adversary cannot learn about the program by watching the memory access patterns (e.g., which address ranges are read in which order).

While TEEs can encrypt the data in use inside RAM and thus keep it confidential,, in cases when the entire data set doesn’t fit into RAM of a single machine, disk, network and other I/O access patterns can still potentially reveal some confidential information.  Thus, ORAM could be a useful addition to a storage system running within TEE to allow it to read encrypted data from external storage without giving away information via the access patterns side channel.

## Ads use cases

### Key value servers

As part of the [FLEDGE proposal](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) two requests are introduced to “trusted” servers that are run by buyers and sellers of ads.  There are limitations on what these servers are allowed to do with the user data that the servers receive, for example, to not correlate users between requests and not have side effects, such as logging.

One application for confidential storage could be to have those servers not retain the keys and values that the browser is requesting.

### Implicit MPC protection

As mentioned above, proposals that rely on MPC can give PIR-like protections as a side effect of their other guarantees.  This holds for the examples linked above as well as where the MPC protocol is auto-generated, as in the [MAhttps://github.com/WICG/privacy-preserving-ads/blob/main/MACAW.md proposal](url).

## Appendix

### Literature and references

**PIR**
* [PIR website](http://www.cs.umd.edu/~gasarch/TOPICS/pir/pir.html)
* [PIR literature survey paper](http://www.cs.umd.edu/~gasarch/TOPICS/pir/mysurvey.pdf)
* [Position paper on why PIR is not practical](https://users.cs.fiu.edu/~carbunar/pir.pdf)
* [Position paper on why PIR is practical](https://www.ndss-symposium.org/wp-content/uploads/2017/09/Usable-PIR-paper-Peter-Williams.pdf)

**TEE**
* [See references here](https://github.com/google/ads-privacy/tree/master/trust-model/tee#appendix-references)

**MPC**
* [See references here](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md)

**ORAM**
* [ORAM and TEEs](https://eprint.iacr.org/2017/549.pdf)
