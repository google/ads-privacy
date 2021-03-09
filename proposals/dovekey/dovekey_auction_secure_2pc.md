
## Disclaimer
_Teams from across Google, including Ads teams, are actively engaged in industry dialog about new technologies that can ensure a healthy ecosystem and preserve core business models. Online discussions (e.g. on GitHub) of technology proposals should not be interpreted as commitments about Google ads products._

_This explainer is neither a cryptographic design document nor a cryptographic 
paper. A more complete pager with more refined crypto details  is coming._

_This explainer does not assume that the audience are crypto experts but rather 
ones who need to evaluate the functionality of the proposed algorithms and its 
advantages and limitations, as well as its performance characteristics._

## Goal

This explainer discusses one possible cryptography design to implement [Dovekey 
auction](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md) 
using Secure Multi Party Computation (MPC) servers. The design delivers a strong 
privacy guarantee and a small set of clearly defined functionality that 
potentially benefits both the user and the ad tech industry. This explainer 
assumes that the audiences are familiar with [Dovekey 
auction](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md). 

## Introduction

The current proposal focuses on one possible cryptography design behind **Step 
6** to **Step 9a** in the sequence diagram in [Dovekey 
auction](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md). 
In those steps, the Dovekey server cluster:

* Evaluates auction eligibility of each bid,
* Runs a combined auction to pick up the winner ad,
* Returns the winning ad back to the browser,
* Processes impression notifications to enforce budget and pacing rules, as well 
  as k-anonymity rules to prevent microtargeting.

This proposal shows that it is possible to implement this set of Dovekey server 
functionalities using Secure 2-Party Computation. In this design, the 
functionality will be carried on by two independent servers which are trusted 
not to collude, and the design guarantees that none of the two servers can learn 
auction eligibility of each bid, the winner ad, or any impression notification. 
While there exist generic methods to implement this functionality, this proposal 
underlines how this functionality can be implemented using the cryptographic 
methods of secret sharing and garbled circuits in an efficient manner.

_We emphasize that it is not the only possible cryptographic design, nor do we 
claim its optimality. We are actively researching how to improve and further 
optimize the design._

![Simplifed Dovekey Server Combined Auction](SimplifiedDovekeyServerCombinedAuctionFlow.png?raw=true "Simplifed Dovekey Server Combined Auction")

## Crypto algorithms

This section provides a basic introduction to some fundamental crypto algorithms 
that the proposed solution depends on.  The following sections will explain how 
the basic crypto algorithms will enable the Dovekey server cluster to support 
concrete ads use cases.

### Additive Secret shares

Assuming that there is a secret message P of integer type. All operations will 
be performed modulo some integer, but for ease of exposition, we do not specify 
the modular operations. To protect its secrecy, we can split it into two secret 
shares P<sub>1</sub> and P<sub>2</sub>, with the constraints that 

* P<sub>1</sub> is a random number;
* P<sub>2</sub> = P - P<sub>1</sub>.

We can then distribute P<sub>1</sub> and P<sub>2</sub> to two non-colluding parties, one secret share 
per party. Without collusion, neither one of the two parties can infer P from 
its own additive secret share.

### Operations over secret shares

The two parties can perform certain operations over secret shares, as discussed 
in the following sections.

#### Addition

Let's assume that there are two secret messages P and Q. Their additive secret 
shares are P<sub>1</sub> and P<sub>2</sub>, Q<sub>1</sub> and Q<sub>2</sub>, respectively, i.e. P = P<sub>1</sub> + P<sub>2</sub>, Q = Q<sub>1</sub> + Q<sub>2</sub>. 
Let's further assume that 

* The first party MPC<sub>1</sub> owns P<sub>1</sub> and Q<sub>1</sub>,
* The second party MPC<sub>2</sub> owns P<sub>2</sub> and Q<sub>2</sub>.

Without colluding with the other party, neither party 
on its own can figure out P, Q or P + Q.

However, MPC<sub>1</sub> may calculate sum<sub>1</sub> =  P<sub>1</sub> + Q<sub>1</sub>, and MPC<sub>2</sub> may calculate sum<sub>2</sub> = P<sub>2</sub> + 
Q<sub>2</sub>. This is done locally, without communicating with each other. It is easy to see that sum<sub>1</sub> and sum<sub>2</sub> are two additive secret shares of P + 
Q. Therefore MPC<sub>1</sub> and MPC<sub>2</sub> just calculated P + Q over secret shares. 

#### Multiplication

Multiplying an additive secret share with a public value can also be done locally by each party 
multiplying its share, for example. On the other hand, multiplying two shared 
values by operating on their share and ending up with only shares of the 
multiplication result (to retain privacy) requires the two servers to 
communicate.

### Garbled circuit

According to Wikipedia, 

>[Garbled circuit](https://en.wikipedia.org/wiki/Garbled_circuit) is a 
>[cryptographic protocol](https://en.wikipedia.org/wiki/Cryptographic_protocol) that enables 
> two-party [secure computation](https://en.wikipedia.org/wiki/Secure_multi-party_computation) in 
> which two mistrusting parties can jointly evaluate a 
> [function](https://en.wikipedia.org/wiki/Function_(mathematics)) over their 
> private inputs without the presence of a trusted third party while retaining 
> privacy throughout the protocol. In the garbled circuit protocol, the function 
> has to be described as a [Boolean circuit](https://en.wikipedia.org/wiki/Boolean_circuit).

The garbled circuit is a collection of computations over Boolean gates mimicking the circuit 
operation with privacy, to produce the circuit's output. For a more in-depth description, see [A Gentle Introduction to Yao's Garbled 
Circuits](https://web.mit.edu/sonka89/www/papers/2017ygc.pdf).

With the above crypto algorithms as building blocks, we can construct a 
candidate crypto design to implement the functionalities described in [Dovekey 
auction](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md). 
The computed values remain shared throughout, and the servers only send their 
shares of output results to the intended receiver (e.g., the browser, when 
computing on behalf of the browser, but they do not send these shares to each 
other). In the rest of the doc, unless otherwise explicitly specified, all 
operations are over additive secret shares, performed with some crypto protocol. 
In addition, in secure 2PC setting, the "Dovekey server" in the 
[diagram](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md#simplification-of-the-flow) 
actually refers to the server cluster with two servers MPC<sub>1</sub> and MPC<sub>2</sub> operated by 
two non-colluding entities. 

## Combined auction

In **Step 6** in the sequence diagram in Section [Simplification of the 
flow](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md#simplification-of-the-flow), 
the Dovekey server starts a combined auction to select winning ads from 

* previously cached conditional bids
* and contextual bids received from the SSP in **Step 5**.

The combined auction includes 3 phases:

1. Determine auction eligibility (**Step 6**)
1. Run combined auction to select the winner (**Step 7**)
1. Select then return the winning ad's creative to the browser (**Step 8**)

The following sections discuss these phases in detail.

### Determine auction eligibility

The browser forms the cache lookup key with a small set of signals, e.g., {`page 
URL`, `browser language`, `coarse location`}. It derives a cache lookup key by 
hashing the signals together so as to reduce the bandwidth and minimize the data 
received by the Dovekey server cluster. The Dovekey server cluster then receives 
the cache lookup key in **[Step 
3](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md#simplification-of-the-flow)**, 
and looks up all cached bids associated with that `cache_lookup_key`. The design 
goal is to handle up to 2<sup>15</sup> = 32,768 cached bids for each `cache_lookup_key`. 

Below, we explain how the Dovekey server cluster computes, for each cached bid, 
the bit is_eligible_for_auction<sub>bid</sub> indicating whether that bid is eligible 
for the auction. This bit will be secret shared between two servers in the 
Dovekey server cluster, i.e., each server will know a random-looking value and 
the sum of both values will be is_eligible_for_auction<sub>bid</sub>. The calculation of 
is_eligible_for_auction<sub>bid</sub>  depends on multiple inputs to be discussed in the 
following sections.

#### Interest Group (IG) membership check

If a cached bid is associated with an IG ID, the bid is an IG bid that is only 
eligible to serve if the browser is a member of the specific IG. Henceforth, if 
the browser is not a member of the bid's IG ID, the Dovekey server cluster should 
eventually obtain is_eligible_for_auction<sub>bid</sub> = 0. Since the browser IG ID are 
sensitive information, we describe below how to implement the private set 
membership using Bloom filters, secret shares, and Garbled circuits.

##### Construct Bloom filter

To concisely represent hundreds or even thousands of Interest Groups (IGs) of which a browser 
may be a member, the browser constructs a probabilistic data structure called 
[Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter). The probabilistic 
data structure requires much less bandwidth to transmit as ad request parameters 
in [Step 3](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md#simplification-of-the-flow) 
than typical data structures, e.g. array or set, may require.

When the probabilistic data structure receives a query

> Is the browser a member of Interest Group X?

The query response may be either

1. **No** with 100% accuracy, i.e. "definitely not".
1. Or **Yes** with a small probability of error, i.e. "most likely yes". The 
   error rate is called the **False Positive Rate 
   ([FPR](https://en.wikipedia.org/wiki/False_positive_rate))**.

There are a few billion browsers on the Internet. Assuming the FPR is 10<sup>-10</sup>, a 
Bloom filter can represent 200 IGs with a bit array of size 9,568 bits (i.e. 
roughly 1.2 KB) with 33 hash functions. Each element in the bit array is either 
0 or 1. For detailed calculation see [Optimal number of hash 
functions](https://en.wikipedia.org/wiki/Bloom_filter#Optimal_number_of_hash_functions).

The browser can create the Bloom filter once, cache it locally until the IG 
membership changes to amortize the bloom filter construction cost. For each ad 
request, the browser randomly splits the bit array into two arrays of additive 
secret shares, bitwise modulo 2. The browser sends each of the two arrays to one 
of the two servers respectively, with combined bandwidth consumption of roughly 
1.2 KB with optimization. 

##### Check membership

To determine auction eligibility, the Dovekey server cluster needs to check the 
IG membership, i.e. whether the browser is a member of the specific IG. First, 
the cluster applies the same set of hash functions over the IG ID and obtains a 
set of locations in the Bloom filter bit array of size `N`. If the Bloom filter 
was not secret shared, all elements at the calculated locations in the Bloom 
filter bit array must be 1 such that the browser can be a member of the IG with 
high probability (i.e. 1 - FPR).

In the secret shared implementation, each of the two servers only holds an array 
of `N` secret shares of the Bloom filter bit array. Therefore, the two servers in 
the Dovekey server cluster execute a cryptographic protocol to determine whether 
the browser is a member of the specific IG, i.e. whether elements in the set of 
locations in the secret Bloom filter are all set to 1. We show in Section [Calculate 
is_eligible_for_auction](#Calculate-is_eligible_for_auction) that such a membership 
test can be done easily using a [Garbled 
circuit](https://en.wikipedia.org/wiki/Garbled_circuit) which computes a `AND` of 
the sums of the `N` bits read by each server and output to each server a secret 
share of the result; this Garbled circuit has `N - 1` gates. Alternatively, one 
may consider the Goldreich-Micali-Wigderson 
([GMW](https://dl.acm.org/doi/10.1145/28395.28420)) protocol to compute this 
functionality.

#### Cross-domain Frequency Capping (FCAP) and Mute-This-Ad (MTA)

On the other hand, when supporting cross-domain FCAP and MTA, the Dovekey server 
cluster should not consider a bid if it belongs to a list of campaigns specified by 
the browser. Once again, this is a membership check setting, where the campaign IDs sent 
by the browser are sensitive, and can be solved using the same approach as 
above.

##### Construct bloom filter

The browser can adopt the same technique as above and construct a Bloom filter 
to support FCAP and MTA, in addition to the Bloom filter to support IG ads. We 
note that is possible to construct a single (slightly larger) Bloom filter to 
support both IG membership and FCAP+MTA.

##### Check membership

The Dovekey server cluster can use the same technique as before to obtain the 
`NAND` of all the bits (instead of the `AND`), and obtain secret shares of a value 
satisfy_FCAP_MTA_rules<sub>bid</sub> 
* equal to `0` if the cached bid is associated with an 
ID present in the Bloom filter to support FCAP and MTA, i.e. the cached bid is 
not eligible for auction due to FCAP and/or MTA rules, 
* and equal to `1` otherwise.

#### Impact of FPR

Due to the FPR associated with FCAP/MTA Bloom filters, some bids will be 
erroneously blocked from participating in the auction for an ad request. For 
buyers, FPR leads to loss of buying opportunity. FPR at 0.1% or less may be 
acceptable from a business perspective. 

In comparison, due to FPR associated with IG Bloom filters, an IG bid may be 
admitted to an auction for a browser that is not a member of the IG, therefore 
may lead to erroneous auction results. To reduce the probability of erroneous 
auction results, we may need to reduce the FPR of IG bloom filters to 10<sup>-10</sup> or 
lower, such that there is a high probability that all IG bids admitted to the 
auction are indeed eligible. Depending on the implementation, it is possible for 
the browser to extract the IG information from the winning ad's metadata, verify 
that the browser is indeed a member of the IG, and therefore may be able to 
detect erroneous action results. 

Bloom filters with FPR at 10<sup>-10</sup> level for IG ads may be significantly costly 
than Boom filters with FPR at 10<sup>-4</sup> level for FCAP/MTA for a few reasons:

1. Lower FPR leads to significantly larger Bloom filters, therefore increasing 
   the browser compute cost and bandwidth to construct and to transmit the 
   Bloom filters.
1. Lower FPR requires significantly more hash functions, which increases the 
   Dovekey server's compute and communication costs to check the Bloom filter 
   membership with crypto protocols.

Assuming the FPR for FCAP/MTA is 0.1%, the optimal number of hash functions for 
the Bloom filter is 13, and the Bloom filter with 40 IDs to block due to 
FCAP/MTA requires 40 \* 19.1 = 764 bits, which is less than 96 bytes.

Real-world data is essential to find the optimal FPR values balancing utility 
and compute/communication cost. We note that various FPR choices do not impact 
user privacy protection.

### K-anonymity

To support k-anonymity which prevents microtargeting, each cached bid is 
associated with a campaign ID, where campaign is one of many possible units to 
apply k-anonymity rules.   
   
Using the [impression 
counting](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md#impression-counting) 
method, the two servers in the Dovekey server cluster can periodically calculate 
satisfy_k_anonymity<sub>campaign</sub> for each campaign, which is a secret message of 
value `0` or `1`. Here again, each of the two servers in the Dovekey server cluster 
holds a secret share of satisfy_k_anonymity<sub>campaign</sub>.

### Budget enforcement and pacing

To enforce budget and pacing rules, the two servers in the Dovekey server 
cluster randomly block a cached bid from participating in the combined auction 
with a carefully chosen probability:

* If the campaign is out of budget, the probability is set to `1`, i.e. always 
  blocks the cached bid from participating in the auction.
* Otherwise,
    * If the campaign is ahead of the delivery schedule, the probability is 
      higher, i.e. the Dovekey server cluster is more likely to block the cached 
      bid from participating in the auction. 
    * If the campaign is behind the delivery schedule, the probability is lower.

The Dovekey server cluster could perform this operation in two phases. 

First, the Dovekey server cluster would periodically compute the 
budget_pacing_selector<sub>campaign</sub> probability. Since secret shares work over the 
integers, budget_pacing_selector<sub>campaign</sub> will be scaled up by a factor of 
`max_range`. To support acceptable throttling precision, `max_range` may need to 
have `7` bits to achieve 2<sup>-7</sup> = 0.8% granularity, in which case `max_range = 127`. 
Since this value is sensitive, this operation will be done over secret shares, 
and the server can perform an arithmetic to Boolean conversion to each secret shares of the bits of budget_pacing_selector<sub>campaign</sub>.

Next, for each ad request and each bid during an auction, the Dovekey server 
cluster will securely perform rejection sampling: it randomly generates a secret 
number uniformly distributed in `[0, max_range]`, and compute whether the secret 
number is less than or equal to budget_pacing_selector<sub>campaign</sub>, which 
indicates whether the bid should be blocked from the auction.

To protect user privacy and business confidential budgeting information, both 
the random number and budget_pacing_selector<sub>campaign</sub> are generated in Boolean 
shares. The comparison between the two values can then be performed using 
[Garbled circuits](https://en.wikipedia.org/wiki/Garbled_circuit) with at most 
`4 * 7 = 28` gates when `max_range = 127`. Alternatively, it is possible to use other 
comparison protocols such as [New Protocols for Secure Equality Test and 
Comparison](https://eprint.iacr.org/2016/544.pdf).

Let satisfy_budget_pacing_rules<sub>bid</sub> denote the comparison result, we have

* satisfy_budget_pacing_rules<sub>bid</sub> is 0 if the cached bid should be blocked 
  from the auction to enforce budget and pacing rules.
* satisfy_budget_pacing_rules<sub>bid</sub> is 1 otherwise.

### Calculate is_eligible_for_auction

For each ad request and each bid, previous sections enumerate the inputs for the 
two servers in the Dovekey server cluster to compute 
is_eligible_for_auction<sub>bid</sub>. We will describe the computation in the following 
sections.

The input to the algorithm includes about 50 Boolean conditions, one bit per 
Boolean condition from each of the two servers, including:

1. 33+ boolean conditions to verify whether the cached IG bid is associated with 
   a IG that the browser is a member of.
1. 13+ boolean conditions to verify whether the cached bid satisfies FCAP and/or 
   MTA rules.
1. satisfy_k_anonymity<sub>campaign</sub>
1. satisfy_budget_pacing_rules<sub>bid</sub>

is_eligible_for_auction<sub>bid</sub> is `TRUE` if and only if all of 1, 3, 4 are `TRUE` and 
2. contains at least one 0, i.e., satisfy_FCAP_MTA_rules<sub>bid</sub> = 1.

We propose to use Garbled circuits to implement the above secure 2PC 
computation. For each ad request and for each cached bid, one of the two servers 
(e.g. MPC<sub>1</sub>) in the Dovekey server cluster constructs a Garbled circuit. MPC<sub>1</sub> 
knows its own secret shares. MPC<sub>1</sub> knows that there is only one possible bit 
pattern that MPC<sub>2</sub>'s secret shares must hold in order for 
is_eligible_for_auction<sub>bid</sub> to become `TRUE`. With such property, MPC<sub>1</sub> only 
needs around 50 gates to construct the Garbled circuit.

### Combined auction

The purpose of the combined auction is to determine the winning bid and clearing 
price, i.e. how much the winner should pay for the impression.

#### Precise bid price

For each cached bid who relies on [dot 
product](https://en.wikipedia.org/wiki/Dot_product) to calculate precise bid 
price for each ad request (see Section 
[Scalability](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md#scalability)), 
the Dovekey server cluster calculates the bid price with dot product between V<sub>bid</sub> and 
V<sub>contextual</sub>. Depending on the privacy and/or business sensitivity of V<sub>bid</sub> and 
V<sub>contextual</sub>, we may decide whether V<sub>bid</sub> and V<sub>contextual</sub> should be protected by 
secret shares:

<!-- TODO: Fix formatting of cells -->
<table>
<tr>
  <th><b>V<sub>bid</sub></b></th>
  <th>V<sub>contextual</sub></th>
  <th>Assumption</th>
  <th>Performance impact</th>
</tr>
<tr>
<td>cleartext</td>
<td>cleartext</td>
<td>Neither V<sub>bid</sub> nor V<sub>contextual</sub> is privacy/business sensitive.</td>
<td>None. Dot product between two vectors in cleartext is trivial. </td>
</tr>
<tr>
<td>cleartext</td>
<td>secret shares</td>
<td>V<sub>contextual</sub> is privacy/business sensitive.</td>
<td>One round of communication between the two servers in the Dovekey server cluster to reconstruct the bid price in cleartext.</td>
</tr>
<tr>
<td>Secret shares</td>
<td>cleartext</td>
<td>V<sub>bid</sub> is privacy/business sensitive.</td>
<td>One round of communication between the two servers in the Dovekey server cluster to reconstruct the bid price in cleartext.</td>
</tr>
<tr>
<td>secret shares</td>
<td>secret shares</td>
<td>Both V<sub>bid</sub> and V<sub>contextual</sub> are privacy/business sensitive.</td>
<td>Most expensive protocol that requires two rounds of communications:
  <ul>
    <li>Round 1 to compute the multiplication between secret shares.</li>
    <li>Round 2 to reconstruct the bid price in cleartext.</li>
  </ul>  
</td>
</tr>
</table>

After calculating the bid price via dot product, the two servers in the Dovekey 
server cluster calculate publisher payout (i.e. "desirability" score in 
[FLEDGE](https://github.com/WICG/turtledove/blob/master/FLEDGE.md#2-sellers-run-on-device-auctions) 
explainer) by applying the SSP-specified pricing rule in secret shares as well.

If the DSP specifies a fixed price for a conditional bid, the SSP's server, in 
lieu of the Dovekey server cluster, can calculate the publisher payout to be 
cached in the Dovekey server cluster. 

#### Select auction winner

First, note that regardless of how the publisher payout is calculated, we assume 
that the two servers in the Dovekey server cluster have publisher payout in 
cleartext for **all bids under consideration** for each ad request, which 
including

1. the contextual bid submitted by the DSP/SSP in **Step 5**.
1. cached bids associated with the ad request's `cache_lookup_key`, whose 
   publisher payout exceeds SSP's applicable floor and the contextual bid's 
   publisher payout.

We explain below how we run the auction. Remember that, for every bid, each of 
the two servers in the Dovekey server cluster possesses a secret share of the 
value is_eligible_for_auction<sub>bid</sub>. 

The first step is for the two servers to **sort** all bids under consideration 
based on the publisher payout. Next, the two servers  iterate through all bids 
under consideration from the highest to lowest publisher payout to calculate 
acc<sub>bid</sub>, which is a running sum of is_eligible_for_auction<sub>bid</sub> for all bids 
traversed so far. This computation can be done locally on their shares and 
independently by both servers, without communicating to each other.

Now, we explain how from the values is_eligible_for_auction<sub>bid</sub> and acc<sub>bid</sub>, it 
is possible to compute is_auction_winner<sub>bid</sub> as  

> is_eligible_for_auction<sub>bid</sub> AND (acc<sub>bid</sub> == 0).  

Indeed, for a bid under consideration, conceptually acc<sub>bid</sub> is the number of 
auction-eligible bids whose publisher payout is higher than the current bid. It will then follow that, for each ad request, is_auction_winner<sub>bid</sub>  is `TRUE` 
for 

* Either one bid, i.e. the winning bid, if the ad request is filled.
* None of the bids, if the ad request is not filled. 

The above process is illustrated in the following table:

<!-- TODO: Fix formatting of cells -->
<table>
<tr>
<th>Bids considered </th>
<th>is_eligible_for_auction  </th>
<th>acc (running sum of is_eligible_for_auction)</th>
<th>is_auction_winner</th>
</tr>
<tr>
<td>{$1.50, …}</td>
<td>0</td>
<td>0</td>
<td>0 (=0 AND 0==0)</td>
</tr>
<tr>
<td>{$1.40, …}</td>
<td>1</td>
<td>0 (=0)</td>
<td>1 (=1 AND 0==0)</td>
</tr>
<tr>
<td>{$1.20, …}</td>
<td>0</td>
<td>1 (=0+1)</td>
<td>0 (=0 AND 1==0)</td>
</tr>
<tr>
<td>{$0.90, …} </td>
<td>1</td>
<td>1 (=0+1+0)</td>
<td>0 (=1 AND 1==0)</td>
</tr>
</table>

The table contains 4 columns. The two servers in the Dovekey server cluster 
conceptually calculate one column at a time, from left to right. All columns 
except the leftmost one are calculated in secret shares that the two servers can't access in cleartext. 

In the above example auction, the $1.40 bid in the second row is the auction 
winner.

### Select and return the winning creative to the browser

Note that both servers will share a vector of bits that sum to the vector of 
is_auction_winner<sub>bid</sub>. This latter vector contains all zeros, and maybe a `1` 
at the location of the winning bid. This is exactly the setting of Private 
Information Retrieval 
([PIR](https://en.wikipedia.org/wiki/Private_information_retrieval)), and the 
Dovekey server cluster can return the winning creative back to the browser 
privately, without learning it.

Specifically, the two servers in the Dovekey server cluster execute information 
theoretic PIR protocol, using is_auction_winner<sub>bid</sub> to select the winning 
creative to return to the browser. The browser receives the winning creative if 
the ad request is filled, or all zeros otherwise.

With classical PIR design, the bandwidth required to transmit the winning ad 
creative back to the browser doubles the size of the largest creative due to the 
two secret shares. There is a potential bandwidth optimization to halve the 
bandwidth required.

## Impression notification

In the proposed 
[architecture](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md#simplification-of-the-flow), 
the impression notification (**Step 9a**) is critical for two reasons:

1. Update budget spent and impression delivered to support budget enforcement 
   and pacing. For details see [Precise and timely budget and pacing 
   control](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md#precise-and-timely-budget-and-pacing-control). 
    
1. Update impression delivered or could have been delivered to support 
   k-anonymity. For details see [Impression 
   counting](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/dovekey_auction.md#impression-counting) 
   .

To support the above functionalities, for each ad request, the Dovekey server 
cluster may log the following information in secret shares after the completion 
of combined auction: 

1. The auction clearing price, i.e. `auction_clearing_price`.
1. is_auction_winner<sub>bid</sub> for all cached bids under consideration.

To calculate `auction_clearing_price` in secret shares, we can use the same 
information theoretic PIR approach as above. For a first price auction, the 
is_auction_winner<sub>bid</sub> shares previously computed can be the PIR selection 
vector. For a second price auction, instead of is_auction_winner<sub>bid</sub>, the 
Doverkey server cluster can compute the vector of values  

> is_eligible_for_auction<sub>bid</sub> AND (acc<sub>bid</sub> == 1)   

as PIR selection vector. This can be done by evaluating another Garbled circuit 
with 30 inputs for 2<sup>15</sup> cached bids, similarly to what is done for selecting the winner ad.

In either case, the two servers in the Dovekey server cluster calculate 
`auction_clearing_price` in secret shares to protect user data and business 
confidential information. At the end of the protocol, each of the two servers, 
MPC<sub>1</sub> and MPC<sub>2</sub>, holds a secret share of `auction_clearing_price`, i.e. 
auction_clearing_price<sub>1</sub> and auction_clearing_price<sub>2</sub> respectively.

Each server may generate an opaque `auction_instance_id` (8 bytes may be 
sufficient) to represent the specific auction. Each server may log its secret 
`auction_instance_id` and return `auction_instance_id` to the browser in **Step 
8**, together with the winning ad. 

After the browser renders the winning ad, the browser passes 
`auction_instance_id`, one for each server, back to the Dovekey server cluster. 
The two servers in the Dovekey server cluster log their respective 
`auction_instance_id` received.

To secure `auction_instance_id` during the transmission i.e. from **Step 8** to 
**Step 9a**, each server may encrypt the `auction_instance_id` with an 
authenticated encryption algorithm for which it maintains the secret key, and 
only provide the encrypted value to the browser (which will appear random).

## Privacy properties

The above secure 2-party computation (2PC) design is a special case of more 
general [secure 
MPC](https://en.wikipedia.org/wiki/Secure_multi-party_computation) design. Under 
the honest-but-curious model, the above secure 2PC design guarantees that 
servers in the Dovekey server cluster don't learn or leak any user data, or 
business confidential data, unless the two servers collude.

## Performance characteristics

Performance is a key factor to determine whether the proposed secure 2PC design 
is practical for production.

### Secure 2PC

Potential Secure 2PC (or MPC) performance bottlenecks include 

* Number of rounds of communication between the two servers
* Number of bytes transmitted in each round
* Compute cost

Some Secure MPC protocols includes

* offline preparation rounds that are input agnostic,
* and online rounds that depend on input.

The proposed secure 2PC algorithm may require 

* 2-3 online rounds that are latency sensitive 
* and a few offline rounds that are latency insensitive. The two servers may 
  complete those offline rounds between **Step 3** and **Step 5**, when the two 
  servers are waiting for the SSP's ad response.

For each online round of communication, the bandwidth required depends on the 
algorithm of choice and the size of the input. As an example, under the 
following assumptions:

1. Garbled circuit is the algorithm of choice to compute Boolean expressions in 
   each round, 
1. The amortized bandwdith cost of Oblivious Transfer extension (OTe) for each Garbled circuit is 
   ~100 bytes.
1. Each bid under consideration for an ad request has about 50 Boolean variables 
   as input to evaluate auction eligibility, and about 30 Boolean variables as 
   input to select the auction winner, i.e. to compute is_auction_winner<sub>bid</sub>.

Assuming 17 bytes per gate, 1 gate per input, 80 gates for the above 80 inputs, 
we will have `17 * 80 = 1.3` KB to transmit the Garbled 
circuits for each conditional bid. 

For each ad request, the bandwidth required to evaluate auction eligibility and 
to select the auction winner is up to 43 MB for up to 2<sup>15</sup> cached bids under 
consideration. If the two servers in the Dovekey server cluster are located in 
the same data center with 10 Gb Ethernet interconnection, it may be possible to 
transmit 43 MB between the two servers in less than 100ms.

Note that the above estimation needs to be verified in live experiments and may 
be optimized further.

### Browser perspective

With the above secure 2PC design, and with the optimization to reduce browser 
bandwidth consumption for ad request and ad response, the number of HTTP 
requests, battery and bandwidth consumption, as well as browser computation 
cost, including the cost to run javascripts to support IG ads, are comparable to 
the status quo.

## Deployment options
### Scale up each server

To support large-scale deployment, each server in the Dovekey server cluster may 
have multiple instances managed by the same party, as shown in the following 
diagram. 

![Server Cluster Configuration](ServerClusterConfiguration.png?raw=true "Server Cluster Configuration")

<a name="fig-1-server-cluster-configuration">Figure 1. server cluster configuration</a>

The instances are organized into

* Serving pool to process incoming HTTP requests, which are latency sensitive.
* Log processing pool to periodically process logs to collect cached bids, 
  compute satisfy_k_anonymity<sub>campaign</sub> and budget_pacing_selector<sub>campaign</sub> 
  then publish resultant snapshots to the serving pool.

To simplify the deployment process, each instance in either one of the two pools 
can be a [Docker](https://www.docker.com/) instance.

The architecture is to support large scale secure 2PC operation. With the log 
processing and snapshot generation/publication design, the two servers are 
guaranteed to operate on the same set of cached bids required by the secure 2PC. 
The serving pool serves 3 types of HTTP requests  

1. From browsers: ad requests and impression notifications. The serving pool 
   will log the auction result and impression notification to support the 
   calculation of satisfy_k_anonymity<sub>campaign</sub> and 
   budget_pacing_selector<sub>campaign</sub>.
1. From SSP: 
    1. request to cache conditional bids. 
    1. Debugging, monitoring and management requests (TBD).
1. From DSP: 
    1. request to update budget/pacing information. 
    1. Debugging, monitoring and management requests (TBD).

### Global deployment view

The global Dovekey server cluster deployment can be shown in the following 
diagram.


![Ideal end state with secure 2PC](IdealEndStateWithSecure2PC.png?raw=true "Ideal end state with secure 2PC")

<a name="fig-2-ideal-end-state-with-secure-2PC">Figure 2. Ideal end state with secure 2PC</a>


In each region, e.g. US East, each one of the two non-colluding parties maintain 
a server denoted by ![Server](ServerIcon.png?raw=true "Server") in [Figure 
2](#fig-2-ideal-end-state-with-secure-2PC), where each ![Server](ServerIcon.png?raw=true "Server") is actually a complete set of Docker instances as shown in 
[Figure 1](#fig-1-server-cluster-configuration), with the exception that the log 
processing pool in [Figure 1](#fig-1-server-cluster-configuration) will only be enabled 
in a small number (e.g. 3) of regions to create and then publish snapshots to 
all regions, while maintaining fallback redundancy.

In [Figure 2](#fig-2-ideal-end-state-with-secure-2PC), communications between servers of two 
parties are limited within each region, where the network bandwidth should be 
abundant, the communication cost and latency should be low.


