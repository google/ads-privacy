# Augury API for TURTLEDOVE

*Teams from across Google, including Ads teams, are actively engaged in industry dialog about new technologies that can ensure a healthy ecosystem and preserve core business models. Online discussions (e.g on GitHub) of technology proposals should not be interpreted as commitments about Google ads products.*

## Background

[TURTLEDOVE](https://github.com/WICG/turtledove) is Chrome’s proposal to enable interest group-based ad targeting in the absence of third party cookies.

In the most general form, there are several hurdles ad technology providers may face in transitioning today’s systems to an on-device TURTLEDOVE bidding implementation:

1. Bidding
    * Applying machine learning models in the browser could be expensive. For example, [TERN explainer](https://github.com/WICG/turtledove/blob/master/TERN.md) mentions that they regularly shipped large models (in the tens of gigabytes) multiple times a day.
2. Brand safety enforcement
    * It is common for advertisers to restrict the placement of their ads, and also for publishers to restrict the ads to show. Similar to bidding, we need to apply all these rules in the browser, which can be expensive and complex to implement.
3. Potential exposure of publishers’, advertisers’, ad tech providers' confidential information and intellectual property
    * The bidding function and rules would be represented as a JavaScript function, which can be inspected by the browser’s user.
4. Billing and auction integrity
    * It’s important for the ad tech providers to be assured of the credibility of submitted bids and auction clearing prices to be used for billing. Getting these assurances is challenging with the on-device bid price computation.

In this document, we present the Augury API that can be offered by an SSP to be used by DSPs as a bidding framework that fits the TURTLEDOVE flow. This API can address these issues, as explained below.

Note that even though we present the Augury API to work with TURTLEDOVE, this API can also be used along with other frameworks such as [Dovekey](https://github.com/google/rtb-experimental/tree/master/proposals/dovekey). Augury API may not be relevant for [SPARROW](https://github.com/WICG/sparrow), where most of the ad selection decisioning happens server-side in the Gatekeeper.

Several assumptions we make:
* Creative rendering is done as suggested in TURTLEDOVE, by prefetching the web bundle associated with each interest group.
* An interest group can be associated with multiple web bundles, identified by a creative ID. On-device bidding logic can choose which creative ID to show to a user.
* Budgeting/pacing and reporting are to be handled separately (out of scope of this proposal).

## Augury API

The essence of the Augury API is for DSPs to submit bids contingent on interest group membership based on the contextual request. These bids provided in the contextual bid response are denoted as *augury bids*, or *augury candidates*. An augury bid needs to contain at least the interest group ID, the creative ID and the bid value. The flow is depicted in the following diagram.

<p align="center"><img src="https://user-images.githubusercontent.com/71041650/98283099-fd754600-1f53-11eb-8cdd-b05807ef4b13.png" width="640"></p>

1. The browser sends a regular contextual ad request to the SSP as described in TURTLEDOVE.
2. The SSP sends a bid request to the DSP.
3. DSP returns bid responses to the SSP. In addition to the regular contextual bids (if any), DSP can also return some *augury bids*.
4. SSP validates the *augury bids* against ad quality checks and publisher brand safety controls and other settings server-side.
5. SSP returns the winning contextual candidate and *augury candidates* that bid higher than the winning contextual candidate to the browser.
6. In the browser, simple on-device bidding logic matches *augury candidates* and creative IDs against browser interest memberships, and admits matching bids to the final on-device auction. Afterwards, the browser proceeds with rendering a winning ad.
    * In addition to interest group membership check, the JavaScript code can also check for other constraints based on user data, such as frequency capping.

By using Augury API, the DSP is only responsible for submitting the predicted bids during contextual ad requests. The SSP is responsible for applying various publisher rules and computing publisher’s payout server-side (as today), and can also optionally provide the on-device interest group matching logic.

Augury API can reduce the technical complexity of implementing TURTLEDOVE across a number of aspects:

**1. Bidding**

Given a predicted interest group, each DSP should be able to reuse most of the existing infrastructure and models to compute the *augury bid* server-side. In other words, the problem is reduced to the prediction of the most likely interest groups given a contextual ad request.

**2. Brand safety enforcement**

Similar to bidding, given a predicted interest group and the associated creative ID, a DSP and an SSP should be able to use the existing machinery to enforce advertisers’ and publishers’ controls server-side, including brand safety and the application of pricing rules to determine auction eligibility and ad technology fee structure to compute publisher’s payout.

**3. Protecting bidding algorithms, publisher and advertiser confidential information**

With Augury API, bid values are computed and rules are applied server-side. The only information that gets returned to the browser is predicted interest groups and bid values.

**4. Billing and auction integrity**

The bid is determined server-side, and hence can be protected by a digital signature of an ad technology provider to be verified by the party handling impression reports. Note that a compromised browser instance (or a non-browser) can still admit the highest-value predicted interest group bid regardless of the actual browser interest group membership, but the potential damage and benefit for malicious actors is considerably less in comparison with computing bid values in the browser.

## Potential API

### Bid Response (OpenRTB)

```jsonc
{
  "id": "Vb8ttXOO",
  "seatbid": [{
    "bid": [{
      "id": "L8QmgECv",
      "impid": "1",
      "price": 2.3,
      "adm": "<img src=\"https://adnetwork.example/ads?id=L8QmgECv&wprice=${AUCTION_PRICE}\">",
      "adomain": "travel.example",
      "cid": "contextual_campaign",
      "crid": "contextual_creative",
      "w": "300",
      "h": "250"
    },
    {
      "id": "9nLHIRIb",
      "impid": "1",
      "price": 7.6,
      "adm": "<img src=\"https://adnetwork.example/ads?id=9nLHIRIb&wprice=${AUCTION_PRICE}\">",
      "adomain": "shoes.advertiser.example",
      "cid": "running_shoes_campaign",
      "crid": "running_shoes_creative",
      "ext": {
        "required_interest_group": "running_shoes"
       },
      "w": "361",
      "h": "203"
    },
    {
      "id": "5vAyHHZS",
      "impid": "1",
      "price": 3.4,
      "adm": "<img src=\"https://adnetwork.example/ads?id=5vAyHHZS&wprice=${AUCTION_PRICE}\">",
      "adomain": "watches.advertiser.example",
      "cid": "sports_watch_campaign",
      "crid": "sports_watch_creative",
      "ext": {
        "required_interest_group": "sports_watch"
       },
      "w": "320",
      "h": "320"
    }]
  }]
}
```

### On-device interest group matching

On-device matching of *augury candidates* can be implemented via the flow described in the [TURTLEDOVE explainer](https://github.com/WICG/turtledove). Augury candidates can be propagated via the contextual response signals to a trivial Augury bidding function. The contextual response signals can take this form:

```jsonc
{
  "interestGroupBids": {
    "sports_watch": 3.4,
    "running_shoes": 7.6,
  }
}
```

Augury bidding function running in the browser could be implemented as a simple lookup of the bidding price from the contextual signals:

```javascript
// Generated Augury bidding function for the sports_watch interest group
function bid(dspContextualSignals, …) {
  return (dspContextualSignals.interestGroupBids?.["sports_watch"]) || 0.0;
}
```

To protect predicted bids from the potential use by other parties against the same interest groups, a DSP or an advertiser ad network should be specified as the reader domain for each interest group. As a consequence, a DSP would receive interest group requests from user browsers and respond with candidate creative(s) for each interest group.

To ease the adoption by DSPs, SSPs supporting Augury API can generate the simple bidding functions illustrated above for their DSP partners based on interest group names. For example, an SSP can expose an HTTP(S) endpoint that returns the Augury bidding function taking the interest group name as a URL parameter, e.g. `https://ssp.example/auguryBiddingFunction?interestGroup=sports_watch`. That way, a DSP would only need to construct the correct SSP bidding function URL when responding to interest group requests.

Thus, in Augury API, the SSPs role could be the handling of interest groups targeting end-to-end: from the contextual bid response to the use of on-device contextual signals in the bidding function.

### Support for multiple creatives for an interest group

As a possible extension to TURTLEDOVE, multiple creatives could be returned to browsers in the interest group response. Each creative can be associated with a creative ID and chosen during the on-device ad selection – for example, returned from the bidding function in addition to the bid value. Augury API implementation can support this use case as well by propagating the chosen creative ID via the contextual signals to the bidding function:

```jsonc
// Contextual response signals
{
  "interestGroupBids": {
    "sports_watch": { "bid": 3.4, "crid": "sports_watch_creative"},
    "running_shoes": { "bid": 7.6, "crid": "running_shoes_creative" }
  }
}
```

```javascript
// Generated Augury bidding function for the sports_watch interest group
function bid(dspContextualSignals, …) {
  return (dspContextualSignals.interestGroupBids?.["sports_watch"]) || 
    null;
}
```

## Generating Interest Group Bids

The core functionality for supporting Augury API includes predicting which interest groups (and associated creative ID) would a browser be a member of given the contextual request, and computing bid values for those.

During each contextual ad request, the [aggregate reporting API](https://github.com/csharrison/aggregate-reporting-api) can be used to record and aggregate the information about which interest groups appeared frequently in the browsers for users that visited a particular publisher website. In addition, the tagging signals (when a user visits an advertiser site) can indicate the popularity of each interest group across a number of dimensions (such as geolocation, device and browser type). The above signals would allow the DSP to make an informed decision to predict which interest groups are more likely to be present in the user’s browser and win the auction.

## Comparison with Dovekey

Both Augury API and Dovekey suggest that DSPs predict interest groups given a contextual request. The difference is that Dovekey caches the predictions inside the KV server, to be queried from the browser, whereas, in Augury API these predicted interest groups and bids are all returned in real time as part of contextual ad response, to be used in the browser.

**Similarity:**

1. Both Augury API and Dovekey require DSP to predict the interest groups given a contextual ad request.
    * The concept similar to [coarse keys](https://github.com/google/rtb-experimental/blob/master/proposals/dovekey/scalability.md), i.e., a single predicted bid that covers multiple interest groups, can also be used on Augury API to improve the coverage in a scalable way.
2. Both Augury API and Dovekey can apply all advertiser and publisher rules server-side, and can help ensure bid authenticity via server-side checking.

**Pros:**

1. Augury API doesn’t need additional trusted servers, and can be implemented within the TURTLEDOVE framework.

**Cons:**

1. SSP needs to send all the predicted interest group candidates to the browser.
    * This may incur non-trivial bandwidth increase.
    * Bid information for many predicted candidates is exposed to browsers.
2. A DSP needs to send many more candidates to an SSP in real time.
    * Predicting all the candidates may consume a non-trivial amount of server-side compute resources and latency budget.
    * One way to address this concern is for each DSP to implement its own caching mechanism, somewhat similar to the KV server as in Dovekey. 
3. SSP needs to apply publisher controls in real time for all submitted *augury bids*, which may consume a non-trivial amount of server-side compute resources and latency budget.

Depending on how critical these concerns are, the SSP and DSP can use both Dovekey and Augury API to complement each other. That is, the SSP can accept a limited number of Augury bids to cover real time information and reduce the KV server usage, while the majority of the bids would still be covered by Dovekey.

## Scalability

In order to cover all possible interest groups, DSP should send bids for all possible interest groups. However, this is infeasible as the number of interest groups may reach millions.

As discussed in Dovekey and in the above section, we propose several mitigations:
1. As in Dovekey, the SSP can accept [coarse keys](https://github.com/google/rtb-experimental/blob/master/proposals/dovekey/scalability.md), where a single bid covers multiple interest groups.
2. Each DSP can cache their predicted interest group bids (for instance, using a subset of bid request properties as a lookup key), to avoid recomputing the interest groups and bids for similar contextual ad requests.
3. In addition to the DSP, the browser can also cache the augury bids for each publisher domain in the first-party storage to reduce the bandwidth cost.

## Compatibility with TURTLEDOVE in-browser Bidding

Augury API is meant to reduce the technical complexity of in-browser bidding – in areas such as bidding, brand safety enforcement, protecting confidential information, and billing and auction integrity – to ease the TURTLEDOVE adoption.

Since Augury API is fully compatible with the TURTLEDOVE, If the DSP prefers the benefits of Augury API, the DSP can use it without an explicit SSP support, provided an SSP supports vanilla TURTLEDOVE on-device auction.

If we believe that both a DSP and an SSP can support in-browser bidding (possibly for some subsets of inventory and demand where it is easier to implement, e.g., for advertisers/publishers with simpler pricing models and brand safety controls), then Augury API can be provided to ease the transition to address the technical complexity of in-browser bidding, or as an option to capture the remaining inventory and demand where server-side bid computation remains critical or more convenient. Which means, the Augury API can live alongside other TURTLEDOVE variants like [TERN](https://github.com/WICG/turtledove/blob/master/TERN.md) and [Outcome-based Turtledove](https://github.com/WICG/turtledove/blob/master/OUTCOME_BASED.md).

## FAQ

**1. What is the limit on the number of bids sent by each bidder?**

The main considerations are the response size to browsers, bid response size to an SSP, and the compute costs incurred by DSPs and SSPs. We are evaluating the practical number of bids that can be evaluated and returned to the browser given those constraints. 

**2. Will returning that many interest group bids to browsers cause bad user experience?**

We expect the relevant browser CPU costs to be small, as it only performs simple intersections. As mentioned above, the network bandwidth could be an issue, which we’re currently evaluating.

**3. Is it a concern that DSP sends multiple interest group bids to an SSP?**

In-browser TURTLEDOVE API can implement access controls (for instance, with specification of [interest group reader domains](https://github.com/WICG/turtledove#browsers-joining-interest-groups) at the time browsers join interest groups), so that interest groups created and accessible by one DSP cannot be accessed or used for ad targeting by other ad tech provider domains. In-browser access controls can help mitigate the concerns about exposing many bids for predicted interest groups: parties other than the DSP would not be able to bid for those same interest groups unless they were permitted at the time the interest group had been joined.

**4. Is it a concern that the SSP sends bids for multiple interest groups to the browser?**

TURTLEDOVE or most of the related proposals suggest that multiple bids for interest group ads to be computed or available in the browser. Given the principle of on-device decisioning, the fact that some bid information gets exposed to the browser seems to be unavoidable. Nonetheless, this issue is less concerning than the one above, as it’s difficult and expensive to scrape bids from browsers at scale.

**5. How to incorporate browser-side signals (e.g., interest group recency) in the Augury API?**

The DSP can send multiple Augury bids for an interest group conditional on the values of bucketed browser-side signals, such as the interest group recency bucket. The DSP can provide its own custom matching predicate to be used in the browser – to choose whether to use the bid computed server-side – that takes into account in-browser signals.
