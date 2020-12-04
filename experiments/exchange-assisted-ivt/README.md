# Experiment proposal: exchange-assisted invalid traffic filtering

*This experiment proposal does not reflect plans of any browser, including Chrome, to support similar means of in-browser ad personalization. While publicly sharing the results of these experiments can lead to productive discussions between ad tech companies as well as browser vendors and privacy advocates on how ad personalization can be supported with stronger user privacy protections, these discussions may not necessarily lead to browsers building support for the use case or the flow similar to the one proposed in this experiment.*

*The experiment may influence, but does not necessarily represent Google’s current or future real-time bidding product plans.*

## Background and existing work

Currently, many RTB bidders rely on certain high-entropy or user-identifying fields for invalid traffic detection, including cookie-based user identifiers or device advertising identifiers, IP address (truncated or full), raw user agent string. For example, a bidder may be able to use a standard list of bots ([such as IAB/ABC International Spiders and Bots list](https://www.iab.com/guidelines/iab-abc-international-spiders-bots-list/)) for filtering out non-human user agents, or build a list of suspicious IP addresses and/or cookie identifiers with unusually high click-through rates and ignore bid requests with those as potentially fraudulent.

Due to their high-entropy nature, these fields present challenges from the privacy perspective.

## Goals

We would like to explore one way that may allow bidders to perform invalid traffic bid-time filtering based on their existing, custom data sources that does not necessarily rely on the presence of high-entropy fields in bid requests. We believe that this approach can strengthen the user privacy protections in real-time bidding while preserving this important existing use case.

## Core idea

A bidder can ask for an exchange to pre-filter bid requests which match field values that a bidder believes to be suspected invalid traffic. For instance, a bidder can provide exchange with a list of IP addresses that it would like to never receive requests for. Or a bidder can upload to an exchange a set of cookie-based identifiers or device advertising identifiers to filter out. An exchange would perform bid request pre-filtering based on the deny-lists or simple rules provided by a bidder periodically.

In addition to privacy improvements, bidders would be able to save on computing costs and network bandwidth, since they wouldn’t have to process requests they believe are invalid. An exchange can optionally report on the aggregated number of bid requests pre-filtered due to blocklists provided by a bidder.

## Potential experiment setup

For an initial experiment, an exchange can offer bidders to pre-filter inventory based on the following fields:
- Full IP address, with specification via a list of CIDR IP ranges to prefilter.
- User agent string, with specification via a list of substrings to match on, or regular expressions to match on.
- Cookie-based identifiers or device advertising identifiers, with bidders providing a set of identifiers to filter out.

A bidder can provide lists of high-entropy fields to filter out to an exchange via an out-of-band API call (exact API spec TBD). An exchange would be expected to protect these lists as confidential as part of the contractual relationships with a bidder. An exchange can also offer bidders to choose bid-time filtering based on well-known lists of bots / sources of high-risk traffic, such as [IAB/ABC International Spiders & Bots List](https://www.iab.com/guidelines/iab-abc-international-spiders-bots-list/).

In an initial phase of the experiment (*Phase 1*), an exchange can continue to send high-entropy fields, such as the `User-Agent` header, normally (subject to applicable privacy controls and regulations). Thus, a bidder would be able to check the correctness of exchange-side prefiltering.

In a later phase of the experiment (*Phase 2*), provided the bidder is comfortable with exchange-side pre-filtering, and a bidder no longer relies on those for IVT detection, an exchange can start redacting some of the high-entropy fields on a small, experimental set of bid requests, to confirm a bidder can continue to transact efficiently without those fields’ presence.

## Assumptions

We assume that an exchange has access to the fields to filter on from the end user’s device, for instance, user agent string, cookie-based identifier or a device identifier.

## Out of scope

Pre-bid invalid traffic detection and filtering performed by third-party providers is out of scope of this proposal. We plan to explore the use of third-party ad fraud detection providers separately.

The use of Trust Tokens and its applicability to detecting invalid traffic in real-time bidding integrations is also out of scope and will be explored later.

## Success metrics

In this experiment, we intend to measure:
- Ability to accurately pre-filter invalid traffic on exchange’s side, using bidder’s custom deny-lists or rules, compared to the current state. Ideally, a bidder should fully retain this functionality in the experiment.
- Reduction in bid request volume sent to a bidder, expressed as a % or QPS, due to a pre-filtering on exchange’s side.
- Change in spend from bidders that use the service for requests on which the high-entropy fields were redacted or replaced by a low-entropy equivalent, for instance, Structured User Agent (instead of raw `User-Agent` string).
- Frequency of deny-list updates.

