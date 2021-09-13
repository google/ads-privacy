# FLEDGE for real-time bidding

*Teams from across Google, including Ads teams, are actively engaged in industry dialog about new technologies that can ensure a healthy ecosystem and preserve core business models. Online discussions (e.g on GitHub) of technology proposals should not be interpreted as commitments about Google ads products.*

## Goals

In this explainer, we would like to suggest a number of approaches to supporting the [FLEDGE](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) in-browser interest group auctions within the traditional real-time bidding integrations between exchanges (SSPs) and bidders (DSPs, ad networks…) – for discussion and iteration on by the RTB ecosystem participants. We hope that starting such a discussion might help align on some possible uniform approaches to support interest group auctions in RTB and make it easier for  bidders and exchanges to adopt FLEDGE.

The explainer is structured in three main parts, each referring to a phase of FLEDGE in-browser auctions: [Before the interest group auction](#before-the-interest-group-auction), [During the in-browser auction](#during-the-in-browser-auction) and [After the auction](#after-the-auction).

### Open questions that are out of scope (for now)

The current explainer does not touch on a number of areas which aren’t yet sufficiently defined by FLEDGE or other Chrome Privacy Sandbox APIs; these will require additional design discussions between browsers and ad tech companies:

- Support for multiple seller auctions: as pointed out in issues [#59](https://github.com/WICG/turtledove/issues/59), [#73](https://github.com/WICG/turtledove/issues/73), and [#202](https://github.com/WICG/turtledove/issues/202), as well as [Prebid Fledge](https://github.com/JoelPM/prebid-td/blob/main/PrebidFledge.md) explainer, FLEDGE API might require additional changes to conveniently support multiple sellers (such as SSPs) in the context of in-browser interest group auctions.
- Aggregated impression reporting – and how it could work for billing and discrepancy reconciliation between exchanges and bidders.

Once there’s more clarity in these areas, we’ll aim to incorporate the relevant design options into this document.

## Before the interest group auction

### Bid request changes: indicating interest group auction support

To differentiate between impression opportunities that support the FLEDGE on-device auction from those that only support the traditional server-side exchange auction, a new enum field called `ae` for “auction environment” can be added as an extension to the Imp object in the OpenRTB bid request to specify which auction environment can be supported by the given impression slot. The `ae` enum can have the following values:

- 0: Traditional server-side auctions.
- 1: Requests with TURTLEDOVE (FLEDGE) support, in which a contextual auction runs on the exchange’s servers and the interest group bidding and the final auction runs in the browser.

```jsonc
{
  "id": …
  "imp": [{
    "id": "1"
    "video": {...}
    "ext": {
      "ae": 0
    }
  }, {
    "id": "2"
    "banner": {...}
    "ext": {
      "ae": 1
    }
  }]
}
```

### Bid response changes

In addition to the contextual bids, a bid response can also be used to specify information relevant for the buyer’s participation in the FLEDGE interest group auctions. The bid response can be updated to support the interest group auctions as follows:

```jsonc
{
  "seatbid": [{
    "bid": [{
      … // Traditional contextual bids
    }]
  }],

  "ext": {
    // InterestGroupBidding object which holds information for running an
    // in-browser interest group auction. 
    "igbid": [{
      // ID of the Imp object of the impression to which 
      // these interest group bidding signals apply to.
      "impid": "1",

      // InterestGroupBuyer object which holds information regarding an
      // interest group buyer for the in-browser auction.
      "igbuyer": [{
        // Domain name of the interest group buyer to participate in the
        // in-browser auction.
        "igdomain": "www.example-dsp.com",
        
        // Optional buyer-specific signals (perBuyerSignals) to pass into
        // the buyer's interest group bidding functions. Can be left empty if 
        // perBuyerSignals are not required by the bidding function.
        "buyerdata": {
          // Example of an arbitrary JSON object defined by the buyer.
          "base_bid_micros": 0.1,
          "use_bid_multiplier": true,
          "multiplier": 1.3,
          "win_reporting_id": "1234567asdf",
          "disallowed_advertiser_ids": ["1234", "2345"]
        },

        // Optional maximum interest group bid price that can be used to 
        // validate the results of the bidding function.
        "maxbid": …,

        // Optional URL that should be fetched if the interest group ad wins 
        // the in-browser auction and results in a billable impression.
        "igburl": "https://dsp.example/imp?auctionid=${AUCTION_ID}"
      }, {
        "igdomain": "buyer2.com",
        "buyerdata": {...}
        "maxbid": ...
        "igburl": ...
      }, {
        "igdomain": "buyer3.com",
        "buyerdata": {...}
        "maxbid": ...
        "igburl": ...
      }]
    }]
  }
}
```

Please note that it is not necessary for the buyer to return a contextual bid, and only the `igbid` information would need to be provided for the buyer to participate in the interest group auction.

#### Interest group auction participation

FLEDGE allows for the seller to explicitly specify [which buyers can participate](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#22-auction-participants) in the on-device interest group auction. An RTB buyer can state their intent to participate in the on-device auction  by returning an `InterestGroupBuyer` object in the bid response with the relevant buyer’s domain name, so that a seller can avoid enabling uninterested buyers for the on-device auction, for reasons such as:
- A buyer might not be comfortable participating in on-device auctions with the given seller
- A buyer might not be running any campaigns targeting inventory received in the contextual request due to advertiser targeting, brand safety restrictions or other business reasons.
- On-device auctions present a technical integration that is quite different from today’s server-side RTB auctions with a different risk profile of potential abuse surfaces. A buyer may want to participate in those auctions only for a subset of requests.

To ensure that only the interest groups actually owned by the buyer can participate in the in-browser auction, an exchange may want to validate that the specified interest group buyer domain name is owned by their demand partner. Additionally, exchanges may need to take measures to avoid processing multiple versions of `InterestGroupBuyer` objects if those can be received from the same buyer via multiple demand paths.

#### Propagating buyer contextual signals (perBuyerSignals)

FLEDGE allows to pass buyer-specific contextual signals into the on-device bidding functions via the [`perBuyerSignals`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#32-on-device-bidding) bidding function input. This data could be constructed by the bidder server-side in the form of an *arbitrary JSON object* during the contextual bid request and sent back to the exchange via the `buyerdata` field as part of the `InterestGroupBuyer` object of the OpenRTB contextual bid response extension. The exchange can then propagate `perBuyerSignals` to the browser and specify as part of the [`auctionConfiguration`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#21-initiating-an-on-device-auction) object if the buyer is eligible for the on-device auction. The browser running the on-device auction will pass the `perBuyerSignals` from the `auctionConfiguration` into the corresponding bidding function based on the buyer’s domain name.

#### Bid integrity protections

Since in the [TURTLEDOVE](https://github.com/WICG/turtledove) proposal, the bid computation and the final auction are to be run locally on the device, this opens up new potential abuse vectors that may affect the integrity of bid values and the final outcome of the auction, making it more difficult to trust on-device auction results (see, for example, [issue #214](https://github.com/WICG/turtledove/issues/214)).

[Interest-group Bid Integrity Support (IBIS)](https://github.com/google/ads-privacy/tree/master/proposals/ibis) proposes a simple approach to mitigate or limit financial risk from these threats by providing a way for bidders to specify a maximum bid price determined server-side and submitted as part of the contextual bid response via the `maxbid` field of the `InterestGroupBuyer` object. This maximum bid price can then be used by the on-device auction or at the time an impression notification is received by a bidder and an exchange to validate the bid calculated by on-device bidding function.

### Determining the contextual winner

An RTB exchange can run its conventional server-side auction to determine the winner among the bids provided in contextual bid responses by demand partners. The information about the contextual winner can then be returned to the browser and compete with the interest group bids in the in-browser FLEDGE auction. In the case no interest group candidate wins, the contextual winning ad can be rendered by an exchange normally.

### Providing creatives to an RTB exchange for scanning

Enforcing ad policies and publisher controls (such as category or advertiser blocks) are some key functions an RTB exchange may perform in addition to running the auction. To enforce ad policies and publisher controls effectively, exchanges may rely on signals about each creative detected by their ad scanning systems. These detected signals could include ad categories, landing page domain and others. Some exchanges may require scanning an ad and knowing these detected signals before a creative is allowed to participate in an on-device auction.

RTB exchanges can provide a creative scanning API that allows buyers to pre-submit their creatives for scanning before bidding with those. The API can accept a render URL – the same render URL as the one a buyer can specify when an interest group is joined – from which a buyer’s interest group creative can be fetched and scanned by the exchange’s ad quality systems.

It may be beneficial to standardize the creatives scanning API interface provided by RTB exchanges to be used for FLEDGE interest group creatives in order to minimize the integration efforts of RTB buyers. The request to upload an interest group creative for scanning from a buyer to an RTB exchange can look as follows:

```http
POST <SELLER_CREATIVE_SCANNING_API_ENDPOINT>
Authorization: Bearer <INSERT_ACCESS_TOKEN_HERE>
Content-Type: application/json
```
```jsonc
{
  // Some creative metadata declared by the buyer.
  "advertiserName": "Test",
  "creativeId": "FLEDGE_Creative_d71e7cd9-8179-4683-ac5c-a203a9c660f",
  "declaredAttributes": [],
  "declaredClickThroughUrls": [
    "https://advertiser.example"
  ],
  "declaredCategories": ["IAB1_6",…],
  // The render URL to fetch an interest group ad from.
  "renderUrl": "https://dsp.example/ads?id=123456"
}
```

The logical response from the seller to the buyer can look as follows:

```jsonc
{
  …
  // Detected ad characteristics such as ad policy compliance 
  // and categories that publishers may select to block in on-device auctions.
  "creativeServingDecision": {
    "detectedClickThroughUrls": [
      "https://testadvertiser.example/books/599041"
    ],
    "declaredCategories": ["IAB1_7",…],
    "servingStatus": "APPROVED"
    "lastStatusUpdate": "2021-07-27T04:14:12.133496Z",
  }
}
```

#### Scanning from losing bids

Exchanges may also want to scan creatives based on the aggregated reports of lost bids. This requires further discussion of the [FLEDGE](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) API design, including information available for the [losing bidder reporting](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#53-losing-bidder-reporting) and how similar information can be made available to sellers, the technical specifications of the [aggregated reporting API](https://github.com/csharrison/aggregate-reporting-api#aggregated-reporting-api), its availability timelines, etc.

#### Scanning from winning bids
Some exchanges may choose to not block creatives from participating in in-browser auctions and serving those until the ad scanning signals are available. While the temporary [event-level reporting](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#5-event-level-reporting-for-now) mechanism is available in FLEDGE, such exchanges could use the render URLs of each winning ad provided to their [`reportResult()`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#51-seller-reporting-on-render) function to initiate creative scanning.

## During the in-browser auction
Common auction signals provided by an exchange
FLEDGE suggests a way for sellers to supply information that will be accessible to all the buyers’ bidding functions and the seller’s ad scoring function in the form of `auctionSignals` as part of the [`auctionConfiguration`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#21-initiating-an-on-device-auction) object. The `auctionSignals` could, for example, include information describing the page context (URL, publisher information, ad restrictions), ad slot (size, format, visibility), or auction behavior (first-price vs. second-price).

For the `auctionSignals` to be useful for the bidding functions provided by different buyers, a standardized data structure might be helpful. One possibility is using a subset of the OpenRTB request and impression objects with fields that are relevant to the on-device bidding.

It is too early to say whether standardized `auctionSignals` would provide added value for buyers over `perBuyerSignals` in practice, and which common signals would be most useful. `perBuyerSignals` would allow each buyer to construct the data [at contextual bid response time](#propagating-buyer-contextual-signals-perbuyersignals) server-side in a format that could be more easily consumed by their on-device bidding functions, for example, by augmenting with extra metadata, performing preprocessing or converting contextual bid request fields to nomenclatures that are tailored to the specific buyer’s bidding use cases.

### Ad quality checks and publisher settings enforcement

RTB exchanges may enforce publisher controls and ad policy compliance on behalf of the publisher within the `scoreAd()` function. When an interest group bid is non-compliant with the required ad quality standards, `scoreAd()` could return a score of zero to disallow a given creative from participating in the auction.

#### Use of trusted scoring signals

To enforce publisher controls and ad quality policies, a number of signals detected by the exchange’s scanning system may need to be used in the on-device auction. One way of making these signals available is using the `trustedScoringSignals`. A buyer can pre-submit their creatives for scanning as described in [Providing creatives to an RTB exchange for scanning](#providing-creatives-to-an-rtb-exchange-for-scanning). When an RTB exchange completes scanning, the signals needed for the enforcement of policies and publisher controls can be loaded into their seller [trusted server](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#31-fetching-real-time-data-from-a-trusted-server) keyed by the creative render URL used for fetching the interest group ads.

#### Using ad metadata from `generateBid`

As an alternative to `trustedScoringSignals`, RTB exchanges could use the `adMetadata` provided by buyer’s `generateBid()`. This may be preferred if an exchange relies primarily on buyer-declared creative metadata for enforcing policy and publisher controls. If the exchange also needs to use signals detected by their scanning system, they can expose an API endpoint for buyers to fetch the interest group ad metadata needed by `scoreAd()`. Buyers can include the exchange-derived signals in each element of the `ads` array when joining an interest group and during the daily refresh calls and then return such metadata from `generateBid()` as part of the bid object. [ROBIN](https://github.com/google/ads-privacy/blob/master/proposals/robin/README.md) describes this data flow in detail and suggests a mechanism for verifying ad metadata within `scoreAd()`.

## After the auction

### Impression counting and billing

#### Possible event for impression counting by exchanges

FLEDGE proposal suggests that the seller-provided `reportResult()` function call [occurs when a winning interest group ad is passed to the Fenced Frame for rendering](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#5-event-level-reporting-for-now). Thus, the `reportResult()` invocation could serve as one possible event which an RTB exchange can use to count impressions.

More conversations between bidders, exchanges and advertising industry bodies will be needed to understand if `reportResult()` invocation meets the requirements of both sellers and buyers for counting impressions and CPM costs, as well as possible mechanisms for minimizing counting discrepancies between buyers and sellers.

#### Bid integrity validation

[Interest-group Bid Integrity Support (IBIS)](https://github.com/google/ads-privacy/tree/master/proposals/ibis) describes a way for buyers and sellers to validate the integrity of an interest group bid by specifying a maximum possible bid price (and a signature to ensure its authenticity) as part of the contextual bid response. These bid price constraints determined server-side during the contextual ad request can be sent to the browser by the exchange as part of the `auctionConfig.sellerSignals`.  The authenticated constraints can then be included in the impression pings sent by seller’s `reportResult()` function and/or the buyer’s `reportWin()` function, which can be used on the server to flag potentially suspicious bids and exclude those from billing and reporting – or mark for further investigation.

#### Attribution of billable impressions to DSP seats

To report to and generate invoices for their partners, exchanges need to attribute impressions resulting from on-device auctions to the correct buyers. RTB bidders may have a complex billing structure: it is common for some bidders to buy on behalf of multiple logical seats on the exchange, e.g. one for each country or each agency client. The limited information available in the `reportResult()` makes it more difficult to support attribution to multiple billable seats in FLEDGE.

RTB exchanges can use the domain of interest group owner to identify the relevant entity to attribute billable impressions to. For instance, exchanges could establish a one-to-one mapping between the interest group owner domain and an RTB bidder that can be billed for impressions delivered via in-browser interest group auctions. During the [temporary event-level reporting phase](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#5-event-level-reporting-for-now) suggested in FLEDGE, RTB exchanges might be able to attribute each impression using the [`interestGroupOwner`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#51-seller-reporting-on-render) field made available to the [`reportResult()`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#51-seller-reporting-on-render). 

Using a separate interest group owner domain for each seat an RTB bidder buys on behalf of for impression attribution may not be ideal. It might result in redundancy and increase network bandwidth costs: a bidder might have to propagate the same copy of `perBuyerSignals` to the browser multiple times, once for each interest group owner domain. Network bandwidth costs may be optimized further with additional design work.

There may be other ways of attributing impressions to a seat, such as the custom reporting dimensions mechanism mentioned in issue [#165](https://github.com/WICG/turtledove/issues/165) or requiring buyers to specify the billing entity for each interest group creative when [pre-submitting them to the exchange for scanning](#providing-creatives-to-an-rtb-exchange-for-scanning). These will require additional design work and discussions between bidders, exchanges and browsers.

#### Impression notifications from exchanges to bidders

Since RTB integrations typically involve payments on impression (CPM) basis, exchanges and bidders often use mechanisms for keeping track of and troubleshooting the potential impression counting discrepancies. One such mechanism described in the [OpenRTB specification](https://www.iab.com/wp-content/uploads/2016/03/OpenRTB-API-Specification-Version-2-5-FINAL.pdf) is the impression or billing notification from an exchange to its bidder partner: a bidder can specify a billing notice URL for each bid they submit (in the `seatbid.bid.burl` field) that an exchange would fetch whenever it records a billable event (typically an impression).

The current FLEDGE proposal describes a temporary [event-level reporting](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#5-event-level-reporting-for-now) mechanism; this section assumes the state when such an event-level mechanism is still available. 

Using such a temporary [event-level reporting](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#5-event-level-reporting-for-now) mechanism, RTB exchanges can provide impression billing notices to the winning buyer of an in-browser interest group FLEDGE auction. Exchanges can use the information made available by browsers to `reportResult()` reporting worklet callback to provide auction outcome-specific signals via the bidder-specified billing notice URL.

Bidders can specify a billing notice URL(s) as part of the OpenRTB contextual bid response – for both contextual bids and interest group ads. For contextual bids, the existing `seatbid.bid.burl` field can continue to be used. For interest group ads, as one possibility, a bidder could specify such an impression billing notice URL *at the bid response level* – as the specific interest group bids are not known at the contextual bid response time and would be determined only during the on-device auction. For example, a new field `InterestGroupBuyer.igburl` can specify a billing notice URL for *any* winning interest group bid of a given buyer:

```jsonc
{
  "seatbid": [{
    "bid": [{
      "burl": ...
    }]
  }],
  "ext": {
    // Information for running an in-browser interest group auction. 
    "igbid": [{
      // ID of the Imp object of the impression to which 
      // these interest group bidding signals apply to.
      "impid": "1",

      // Information regarding an interest group buyer for the in-browser auction.
      "igbuyer": [{
        // Domain name of the interest group buyer to participate in the
        // in-browser auction.
        "igdomain": "www.example-dsp.com",
        
        // The URL that should be called if the interest group ad of this buyer
        // wins the on-device auction and results in a billable impression.
        "igburl": https://dsp.example/impression?id=123456&auctionid=${AUCTION_ID}&interestgroupname=${INTEREST_GROUP_NAME}&renderurl=${RENDER_URL}&auctionprice=${AUCTION_PRICE}
      }]
    }]
  }
}
```

The URLs that bidders may specify can contain macros that, when replaced by an exchange, provide additional information about the auction. Exchanges may want to continue to support the following existing OpenRTB macros:
- `${AUCTION_ID}` should be supported so that bidders can associate an impression tracking URL to the original bid request.
- `${AUCTION_PRICE}` should be supported so that bidders can learn the winning bid price (from exchange’s point of view), which is determined during the in-browser auction. For interest group ads, sellers could derive this value from the `bid` field in the [`browserSignals`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#51-seller-reporting-on-render) object during [`reportResult`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#51-seller-reporting-on-render) reporting calls.

In addition, among  [`browserSignals`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#51-seller-reporting-on-render) that sellers can access while handling [`reportResult`](https://github.com/WICG/turtledove/blob/main/FLEDGE.md#51-seller-reporting-on-render) reporting calls, `interestGroupName` and `renderUrl` may be of interest to buyers for impression tracking. The macros `${INTEREST_GROUP_NAME}` and `${RENDER_URL}` could be supported by exchanges, based on the information browsers may make available to `reportResult`, to give buyers more information about the winning ad in the impression tracking URL. 

An example of a bidder-provided impression tracking URL for interest group ads may look as follows: `https://dsp.example/impression?id=123456&auctionid=${AUCTION_ID}&interestgroupname=${INTEREST_GROUP_NAME}&renderurl=${RENDER_URL}&auctionprice=${AUCTION_PRICE}`.

### Reporting interest group ad clicks to an exchange

RTB exchanges often need the information about ad clicks in order to report click metrics to publishers, as well as fight malvertising, fraud and abuse. In today’s RTB integrations, a buyer’s creative is typically provided as an arbitrary HTML code. Since a buyer controls the click-through event in the provided HTML, exchanges often ask RTB demand partners to insert a click macro, which they replace before rendering a winning ad on a device with their own click beacon URL.

In contrast with today’s RTB integrations, in the current [FLEDGE](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) proposal, the seller's logic does not get to inspect or augment the buyer’s creative code from the `renderURL` before it is shown. Furthermore, a winning interest group creative is expected to be rendered within a Fenced Frame that cannot access the page contextual information – making it difficult for a seller to provide its query-specific click beacon URL to the creative.

[Fenced Frames Ads Reporting](https://github.com/WICG/turtledove/blob/main/Fenced_Frames_Ads_Reporting.md) proposal suggests that ads rendered within Fenced Frames could notify buyers and sellers in FLEDGE auctions of various ad events, such as clicks. Buyers and sellers could register beacons to receive these notifications via [`registerAdBeacon`](https://github.com/WICG/turtledove/blob/main/Fenced_Frames_Ads_Reporting.md#registeradbeacon) call. An ad rendered within the Fenced Frame could use [reportEvent](https://github.com/WICG/turtledove/blob/main/Fenced_Frames_Ads_Reporting.md#reportevent) call to let the buyer’s, the seller’s or both reporting worklets know of a click.

A standard event type can be used with this API for communicating clicks between interest group buyer ads and sellers. 

An RTB exchange in the role of a FLEDGE seller could register a beacon for the event of type `click` and expect its demand partners to make `reportEvent` call from within the ad rendered in a Fenced Frame whenever a user clicks on an ad: 

```javascript
navigator.reportEvent({'eventType': 'click', 'destination': ['seller'…]|)
```

An RTB exchange’s registered beacon could then serve the same role as today's click tracker URL provided via a macro replacement into ad HTML snippets.

