## Disclaimer

_Teams from across Google, including Ads teams, are actively engaged in industry dialog about new technologies that can ensure a healthy ecosystem and preserve core business models. Online discussions (e.g. on GitHub) of technology proposals should not be interpreted as commitments about Google ads products._

## Introduction

[DOVEKEY](https://github.com/google/ads-privacy/tree/master/proposals/dovekey), 
a modification of [SPARROW](https://github.com/WICG/sparrow), proposed to have 
the Gatekeeper act as a simple key-value cache server.

Here we present multiple enhancements to DOVEKEY to address scalability concerns and 
allow it to include an auction.  The auction then allows the implementation of 
several common ads features inside the Dovekey Server:

1. Cross domain [frequency 
   capping](https://github.com/w3c/web-advertising/blob/master/support_for_advertising_use_cases.md#frequency-capping).
1. Mute-this-ad - this is a feature for users that allows the disabling of a 
   certain ad.
1. Micro-targeting prevention.
1. Precise and timely budget and pacing control.

## Motivation

These changes improve DOVEKEY in a number of ways:

User benefits:

1. They reduce the ad response size to the browser, by returning only one ad, 
   and therefore the amount of network bandwidth needed.
1. Cross domain frequency capping and Mute-This-Ad improves users' ad 
   experience.
1. Micro-targeting prevention enhances privacy.
1. More computation shifts to the server and reduces browser resource usage.

Advertiser and Ad tech benefits:

1. The scalability of the number of cached bids necessary was the key problem 
   identified in feedback, which is addressed in this explainer
1. Cross domain frequency capping and Mute-This-Ad reduces advertising budget 
   waste and also improves advertisers ROI.
1. Finer control of budgeting and pacing to prevent overspend and underspend, 
   enables Ad Tech Partners (ATPs) to satisfy delivery goals.

## Scalability

In Dovekey, each cached bid is associated with a fixed bid price specific to the 
Interest Group (IG). The fixed price is not customizable per ad request. To 
increase bid accuracy, Dovekey has to form a large volume of cache keys, one for 
each granular inventory slice. Such design leads to the combinatorial explosion 
in the number of cache keys.

Here we propose having the bidding function be the dot product of the IG and 
contextual data.  That allows us to reduce the number of keys to linear.

To do that, each cached bid is allowed to also have a vector of floating point 
numbers stored with it, denoted by V<sub>bid</sub>.  The 
data inside V<sub>bid</sub> is not specific to any one user 
but is specific to an IG.

For each contextual ad request, the ATP returns one single vector of floating 
point numbers, denoted by V<sub>contextual</sub> to the browser.  
V<sub>contextual</sub> may be derived from contextual signals (e.g. 
page URL) as well as FLoC id.

The final bid price, calculated by the Dovekey Server, for the cached bid is the 
dot product of V<sub>bid</sub> and V<sub>contextual</sub>.  This allows an ATP to upload a single cached bid to a Dovekey 
Server that can result in different final prices depending on the contextual 
information.

We don't believe that this alters the privacy protections.

## Combined auction

As a reference, this is the [overall Dovekey 
flow](https://github.com/google/ads-privacy/blob/master/proposals/dovekey/README.md#browser-requests-interest-group-bids-from-kv-server). 
 The second half, **in bold**, is updated to support the additional 
functionality:

1. The browser sends a regular contextual ad request as described in TURTLEDOVE.
1. The SSP returns the winning contextual ads, along with derived contextual 
   signals (from SSP and DSP separately.) The final contextual signal is formed 
   as the concatenation of those from a DSP and an SSP.
1. The browser constructs the cache lookup key based on the contextual signal 
   and sends the cache lookup key and the interest-group IDs (ig\_id) in a 
   request to a Dovekey Server.
    1. **It also includes a list of any conditional ads whose conditions should 
       be evaluated in the Dovekey Server. Examples of supported conditions 
       include:**
        1. **Interest Group membership.**
        1. **Whether the ad is blocked due to cross domain frequency capping or 
           Mute-This-Ad.**
        1. **Whether the ad ****campaign**** has budget left, or is over- or 
           under-delivering, and should be throttled properly.**
        1. **Whether the ad campaign satisfies k-anonymity rules to prevent 
           micro-targeting.**
1. **The Dovekey Server determines auction eligibility of the cached bids and 
   runs an auction to determine the winning bid.  The winning bid is returned to 
   the browser.**
1. **The browser runs an auction between the interest-group candidate fetched 
   from the Dovekey Server and the winning contextual ad.**

User data (e.g Interest Group membership) sent to the Dovekey Server in Step 3 
cannot be in plaintext because it may be possible to use them to track users 
across multiple requests.  There are several possible technical solutions to 
this, see below for details.

### Simplification of the flow

For each ad slot, the above flow requires the browser to send two requests 
sequentially and receive two responses in Step 1 and Step 3 respectively. The 
two request/response roundtrips increase the overall latency, as well as mobile 
device battery and bandwidth consumption over status quo. Furthermore, the 
functionalities proposed in this explainer don't apply to contextual ads, which 
may lead to inconsistency in user experience, privacy protection and ads 
products functionality. To address the above issues, we can further simplify the 
Dovekey flow as follows.

![Simplifed Dovekey Server Combined Auction](SimplifiedDovekeyServerCombinedAuctionFlow.png?raw=true "Simplifed Dovekey Server Combined Auction")

**Explanation**:

1. The SSP's ad tag in the browser prepares the contextual ad requests as it 
   does so in the original TurtleDove and Dovekey proposal.
1. The browser encrypts the contextual ad request so that only the SSP may 
   decrypt.
1. The browser bundles the IG requests and encrypted contextual ad request 
   together and sends the bundle to the Dovekey server.
1. The Dovekey server forwards the encrypted contextual ad request to the SSP, 
   then waits for the contextual ad response from the SSP.
1. The Dovekey server receives the ad response and caches conditional bids embedded in the contextual ad 
   response locally. Those cached bids will be considered for future IG ad 
   requests.
1. The Dovekey server applies all the proposed functionality across all cachied 
   bids that might be eligible for the current IG request, as well as the 
   contextual ads received from the SSP.
1. The Dovekey server runs a combined auction between cached IG bids eligible 
   for auction and contextual ad received from the SSP in contextual ad 
   response.
1. The Dovekey server returns the winning bid and snippet to the browser and the ad tag (maybe in the form of an opaque handle) for 
   rendering.
1. When the browser renders the winning ad snippet, the browser sends an 
   impression notification back to the Dovekey server to support budget, pacing 
   and microtargeting prevention. For details see later sections.

At architecture level, the requests/response flow among multiple entities are shown below

![Dovekey Combined Auction Architecture](DovekeyCombinedAuctionArchitecture.png?raw=true "Dovekey Combined Auction Architecture")

In the event of Dovekey server production outage, the browser can send the 
contextual ad request directly to the SSP to show only contextual ads to the 
browser.

### Overview

The Dovekey Server has a list of ads, each of which has the publisher payout (or 
other metric based on which the auction is performed), that're keyed by the 
cache lookup key for the current IG request.

A first price auction is done by sorting all bids by the publisher payout and 
then iterating through that list from the highest publisher payout to the lowest 
to look for the first ad that is eligible for auction. Other auction ranking 
adjustments aren't covered here.  The Dovekey server evaluates the eligibility 
of each cached bid based on either user data (e.g. IG membership) received in ad 
request, or variables/counters maintained by the Dovekey server, e.g. remaining 
budget of a campaign whose values are derived from sensitive user information 
(e.g. an IG ad is shown to a particular user).  

### Mute-this-ad (MTA)

This is the simplest of the use cases to support:

1. Each ad includes a unique MTA ID (e.g. campaign id, or a term in a taxonomy) 
   when it is downloaded to the browser.
1. When the user mutes an ad, the browser adds the unique MTA id to a local 
   store.
1. Requests to the Dovekey Server include the list of MTA IDs that have been 
   muted.
1. For each ad request, the Dovekey server filters all cached bids associated 
   with the MTA IDs so that they can't participate in the auction and cannot 
   win.
    1. Note that a browser could implement per-interest-group MTA by not calling 
       the Dovekey Server at all but the current proposal allows support of 
       campaign-level MTA.

### Frequency capping (FCAP)

This flow is similar to the mute-this-ad flow:

1. Each ad uses a new browser API to report:
    1. The unique FCAP ID, which could be the campaign id.
    1. The frequency capping rules for this campaign.
1. The browser stores these if they're for a new FCAP ID, using a new browser 
   API.  If they're for a previously seen FCAP ID then it updates a counter of 
   how many times ads associated with the same FCAP ID have been seen.
1. If the FCAP ID has reached the frequency cap limits then the browser adds it 
   to a list of FCAP IDs that should be filtered.
1. Filtering happens in the same way as MTA, i.e. requests to the Dovekey Server 
   include the list of FCAP IDs that have been muted. For each ad request, the 
   Dovekey server filters all cached bids associated with the FCAP IDs specified 
   in the ad request so that they can't participate in the auction and cannot 
   win.

### Precise and timely budget and pacing control

We add a basic impression reporting mechanism for the browser to update counters 
in the Dovekey Server that can then be used for budgeting and pacing.

The counters effectively provide a filter for the Dovekey Server to apply based 
on whether a cached bid has run out of budget, or whether the cached bid is 
behind or ahead of its delivery goals specified by the DSPs.  We can support 
having the budgeting data about an ad be in cleartext if it's not sensitive or 
inside an opaque secret share if it is.

If the browser selects the Dovekey Server's ad and renders it, the ad, in 
association with the browser, reports an impression to the Dovekey Server.  This 
doesn't have to be realtime, may have noise added, and can be made opaque 
(depending on the server architecture) with techniques like secret sharing - 
we'll be following this up in another explainer.  Based on the impression 
notification, the Dovekey Server updates the budget and delivery counters 
indicating the actual ad delivery process. 

### Micro-targeting prevention

We want to extend the basic auction model to include the ability to filter out 
ads that are targeted to too few users - this objective is similar to the one 
proposed by [RTB House's Outcome Based 
TURTLEDOVE](https://www.google.com/url?q=https://github.com/WICG/turtledove/blob/master/OUTCOME_BASED.md&sa=D&ust=1606835294687000&usg=AOvVaw0uDZpkAM3HXJ-00hXo3oBB). 
 

#### Impression counting

This builds on the budgeting and pacing proposal above, the impression 
notification mechanism in particular.

Until an ad satisfies the k-anonymity rule for micro-targeting prevention, the 
ad can't be shown to any user. Without showing the ad to any user, how do we 
know whether it satisfies the k-anonymity rule?

To answer the above question, the Dovekey Server runs a counterfactual auction - 
the same as the actual auction except that the counterfactual auction doesn't 
apply the k-anonymity rule.  This design defines k-anonymity as at least k 
unique browsers could have seen an ad in a certain timeframe.

The Dovekey Server sends the winners of both auctions to the browser. If the 
browser's auction chooses the Dovekey Server's ad as the final winner and 
renders the ad, the browser sends an impression notification to the Dovekey 
Server to update the k-anonymity counters (in addition to the budget and 
delivery counters) associated with the winner ads in both the actual and 
counterfactual auction.  The browser only requests the Dovekey server to update 
the k-anonymity counter for an ad if it hasn't already requested to do so for 
that ad in the k-anonymity lookback window.  (The browser stores which ads it 
has sent reports for.)

In this proposal, the impression notification plays the double duty to update 
both the budget and delivery counters, as well as the k-anonymity counters. Such 
double-duty reduces the QPS on the Dovekey server. Furthermore, the browser may 
send the impression notification to the Dovekey server piggy-backed on the next 
ad request. Such piggy-back further reduces the QPS on the Dovekey server and 
the network connection cost on the browser side. 

## Privacy model

These proposals are applicable in a variety of server-trust models and so this 
explainer doesn't go into the implementation details of how those will differ, 
e.g. between a single trusted server, Secure Multi-Party Computation (MPC) 
servers, etc.

### User privacy

All additional user data needed for these proposals is stored in the browser.

The inclusion of k-anonymity checks in the Dovekey Server prevents 
micro-targeting.

The biggest change in these proposals is the addition of an impression reporting 
mechanism back to the Dovekey Server.  This needs further discussion but we 
believe it to be practical because:

1. No user identifier is necessary.
1. Cryptography (or differential privacy noise) can be added to the channel to 
   protect user privacy.
1. The reporting does not have to be realtime - it depends on what accuracy we 
   want for budgeting, pacing, and micro-targeting prevention.

### Ad bid data

In DOVEKEY the bid price for each ad is not hidden and is returned to the 
browser for the auction.  In this proposal the Dovekey Server would make use of 
the bid price as well in order to select the winning ads with the highest 
publisher payout, this should not be a big difference.

If the budgeting and pacing targets for an ad turn out to be sensitive business 
information, the Dovekey server may protect such information with Secure 
Multi-Party Computation design.

#### List of ads to filter

The user data that's sent from the browser to the Dovekey Server to determine 
the auction eligibility of cached bids, e.g. users' Interest Group membership 
could be used as an identifier for a user and so it needs to be passed in such a 
way as to make that impossible.

There are various mechanisms for that which we're actively investigating, 
including the use of [secret shares](https://en.wikipedia.org/wiki/Secret_sharing). The solution chosen 
depends on the way that the Dovekey Server is set up and the trust model used. We plan on publishing future explainers with crypto details.
