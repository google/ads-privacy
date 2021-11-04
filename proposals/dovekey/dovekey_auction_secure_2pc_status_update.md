# Status update: Dovekey Auction with Secure 2-Party Computation (2PC)

## Disclaimer
_Teams from across Google, including Ads teams, are actively engaged in industry dialog about new technologies that can ensure a healthy ecosystem and preserve core business models. Online discussions (e.g. on GitHub) of technology proposals should not be interpreted as commitments about Google ads products._

_This explainer is neither a cryptographic design document nor a cryptographic 
paper. This explainer does not assume that the audience are crypto experts but 
rather ones who need to evaluate the functionality of the proposed algorithms 
and its advantages and limitations, as well as its performance characteristics._

# Goal

In March 2021, Google Ads published the [Dovekey auction with secure 
2PC](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md) 
explainer, which outlines a possible Multi-Party-Computation 
([MPC](https://en.wikipedia.org/wiki/Secure_multi-party_computation)) design for 
privacy preserving ads selection process. Since then, we have prototyped the 
proposed architecture to evaluate its technical feasibility. This explainer 
summarizes some of the findings in the prototype.

# Scope of the prototype
## Features

The prototype implemented a subset of the features and crypto algorithms 
discussed in the [Dovekey auction with secure 
2PC](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md) 
explainer, including:

* 1st price auction to select the winner (See [Combined 
  auctions](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#combined-auction) 
  for detail)
* [Interest Group membership 
  check](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#determine-auction-eligibility)
* [Cross-domain Frequency Capping (FCAP) and Mute-This-Ad 
  (MTA)](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#cross-domain-frequency-capping-fcap-and-mute-this-ad-mta)
* [K-anonymity check 
  ](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#k-anonymity)
* [Return winning creative to the browser via 
  PIR](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#select-and-return-the-winning-creative-to-the-browser), 
  where PIR stands for [Private Information 
  Retrieval](https://en.wikipedia.org/wiki/Private_information_retrieval).
* [Precise bid price 
  calculation](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#precise-bid-price) 
  per ad request via [dot product](https://en.wikipedia.org/wiki/Dot_product).

With the following exceptions:

* The prototype fully implemented the server side crypto algorithm, but mocked 
  client side or DSP/SSP side logic. DSP/SSP stand for Demand Side Platform 
  (e.g. ad network) and Supply Side Platform (e.g. Google Ad Manager) 
  respectively.
* [Budget enforcement and 
  pacing](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#budget-enforcement-and-pacing) 
  feature is under development.
* [Impression 
  notification](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#impression-notification) 
  is postponed because it is not on the critical ad serving path and is not 
  latency sensitive.

## Out of scope

There are many technical and non-technical risks involved in trusted servers 
based on MPC. The focus of this prototype is to assess and mitigate most, but 
not all, of the critical technical risks. Network latency optimization is not in 
scope.

# 3-stage ads selection process

The prototype conceptually implements a 3-stage ads selection process, as shown 
in the following diagram. In the diagram, L, M, K and 1 are the number of 
candidate bids flowing through the corresponding pipe.

![3-stage ads selection in Dovekey auction prototype](3StageAdsSelectionProcess.png)
<a name="fig-1-ads-selection-process">Figure 1. 3-stage ads selection in Dovekey auction prototype</td>

It turns out that the details of the 3-stage ads selection process has profound 
impact on the revenue recovery potential and the computation/latency cost, as 
well as the communication cost between the two parties in the MPC setup.

## Stage \#1 cache lookup 

Logically the inputs to **Stage \#1** are all the conditional bids, conditioned 
on user data (or its derivatives) not available on Ad Tech Providers' (ATPs) 
servers. Those conditional bids are cached on the MPC cluster. The value **L** could 
be in the range of 10s of millions depending on the ATP use cases and the 
signals that may comprise the contextual cache key. The value L determines the 
MPC cluster's storage cost.

The output of Stage \#1 includes **M** cached conditional bids associated with the 
cache lookup key, which is formed with selected signals in the contextual ad 
request (See 
[link](dovekey_auction_secure_2pc.md#determine-auction-eligibility) 
for detail). For the prototype M is 32,767.

## Stage \#2 bid price computation and floor enforcement

For each of the M cached conditional bids from the previous stage, the MPC 
cluster

1. computes the precise bid price for the current request with the dot product 
   method.
1. Apply applicable floors to filter out bids whose bid prices are below 
   applicable floors or the global minimum bid price (e.g. $0.01 in the US), or 
   the highest contextual bid.

The output of Stage \#2 include top-K cached conditional bids with valid bid 
prices. For the prototype,

* K can be up to ~16,000 depending on the random bid prices generated. In some 
  benchmark tests, we capped the K value to fixed numbers, e.g. 15,000, via a 
  command line flag, to obtain reproducible benchmark results from downstream 
  crypto algorithms.
* The vector dimension for the dot product to calculate precise bid price is 
  128. A dimension of 32-64 may be sufficient for production, in which case we 
  can further reduce the computation cost for the bid price calculation.

All computation in this stage is in cleartext in the current design, which is 
permissible because the computation doesn't rely on any cross-site user signal.

## Stage \#3 auction 

In this step, the MPC cluster relies on MPC algorithm to select the final 
auction winner from the top-<!-- No converter for: EQUATION --> candidates by 
leveraging full fidelity signals not available to the ATPs' servers, including 
user's Interest Group (IG) membership, FCAP/MTA, budget/pacing and k-anonymity.

Due to the inherent computation cost of MPC crypto algorithms, this stage 
requires significantly more computation per ad candidate, compared to all 
previous stages.

In the prototype, the size of the winning ad is assumed to be randomly 
distributed between 4KB and 8KB, which are roughly the 50% and 90% of raw HTML 
snippet sizes distribution in production (though not counting assets these ads 
may download), respectively. 

In the future when winning ads are rendered within fenced frames without network 
access, the winning ad bundle must include all creative assets necessary for 
rendering, in which case the winning ad size distribution may change and the 
change may require additional performance analysis and optimization.

## Impact to Revenue/Serving cost

Unsurprisingly, when increasing L, M and K, 

1. The estimated Interest Group Ads revenue in prototype increases 
   asymptotically,
1. Yet the serving cost, dominated by the crypto algorithm, increases linearly. 
   The serving cost here includes the computation cost, serving latency, as well 
   as communication cost between the two parties in the MPC setup.

   
There are rooms to further improve the revenue and/or reduce the serving cost. 
For example,

1. Probabilistically calculate the dot product for bid price with software 
   packages like  [fbgemm](https://github.com/pytorch/FBGEMM).
1. More efficient MPC algorithm.

# Serving cost measured
## Computation cost

With the above setup and parameters, we observed that, for each ad request, the 
prototype takes on average combined 115ms for both parties in the MPC setup to 
complete all processing on an Intel Skylake Xeon 3GHz CPU in single-threaded 
mode, without accounting for the network communication latency/cost. The 
performance is achieved via a combination of optimization at all levels, 
including business use case, architecture, crypto algorithm, implementation, and 
low level assembly-like programming with Intel 
[intrinsics](https://en.wikipedia.org/wiki/Intrinsic_function).

## Communication cost between two parties

In the prototype, for K = 15,000, the total online communication between the two 
parties is 18.76 MB, which is linear to the value of K and the number of [hash 
functions](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#construct-bloom-filter) 
in the bloom filter for IG, FCAP/MTA membership check. We are exploring 
additional optimizations to reduce the bandwidth consumption.

## Serving Latency 

The overall MPC architecture and crypto algorithm require 3 types of Remote 
Procedure Calls ([RPCs](https://en.wikipedia.org/wiki/Remote_procedure_call)) 
between the two parties in the MPC setup.

1. An asynchronous RPC, independent of any ad request, sets up the states of the 
   two parties properly. The RPC cost can be amortized across ~1,000 ad 
   requests.
1. While the two parties are waiting for the contextual ad response from the 
   SSP, the two parties complete one RPC to initiate Random 
   [OTe](https://en.wikipedia.org/wiki/Oblivious_transfer). The RPC takes less 
   than 10 milliseconds assuming the two parties are co-located in the same 
   modern datacenter,  while the SSP typically takes a fraction of a second to 
   produce the response.
1. After the SSP's contextual ad response is received, the two parties in the 
   MPC setup perform one last RPC call to complete the auction process and to 
   send the winning ad snippet back to the browser.

The first two RPCs don't directly contribute to the overall ads serving latency. 
Compared to the last RPC, the computation cost and bandwidth consumption due to 
the first and second RPC is negligible. The net latency increase introduced by 
the MPC cluster over status quo is due to the last one of the 3 RPCs between the 
two parties. To reduce the serving latency, we need to reduce the latency due to 
computation and communication associated with the last one of the 3 RPCs.

We may perform the computation with multi-threading. The ​​[Garbled 
circuit](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction_secure_2pc.md#garbled-circuit) 
computation accounts for roughly 90% of all computation cost. It is highly 
parallelizable because the garbled gates computation for one cached bid is 
largely independent of the computation for other cached bids. Initial 
benchmarking demonstrated that, by enabling multi-threading for Interest Group 
and FCAP/MTA membership check, with 16 threads, we can reduce the overall 
computation latency for both parties in the MPC setup by half compared to single 
threaded implementation, i.e. from ~115ms to ~55ms.

The latency introduced by communication between the two parties is highly 
dependent on the network capacity and condition. Optimizing the network setup to 
reduce the latency due to communication is out of the scope of the prototype.

# Summary

The prototype suggests that the MPC algorithm and associated optimization as 
implemented is mature enough to support selected ads use cases, with serving 
costs that are high but feasible.
