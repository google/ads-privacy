# Ad Manager’s Initial Testing of FLEDGE With Multiple Sellers
## Goal
We would like to outline how Ad Manager will support initial testing of [FLEDGE](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) with multiple sellers (i.e. exchanges) that might be working with Ad Manager publishers during Chrome’s origin trial. 

## Context 
To enable early FLEDGE testing for different exchanges on Ad Manager inventory during Chrome’s origin trial, Ad Manager will offer a lightweight API for exchanges to specify their auction configurations via the Google Publisher Tag (GPT) API. 

The API presented here is intended to be only a temporary mechanism to enable testing during the origin trial, and does not necessarily reflect Google’s future product plans for the support of multiple sellers in FLEDGE auctions on Ad Manager publisher inventory.

## Technical details
To enable different exchanges to test FLEDGE in Origin Trials with publishers that use Ad Manager as an ad server, Ad Manager will add an experimental method to the GPT API, `Slot.setConfig({ componentAuctions: [{configKey, auctionConfig}] })`. Other sellers (i.e. exchanges) that work with a publisher can call this method before the publisher initiates an ad request to Ad Manager. GPT would collect all provided auction configurations, use those to populate the `componentAuctions` field of the top-level `auctionConfig`, and commence the top-level auction if the ad request is enabled for FLEDGE.

Sellers might need to update their previously provided auction configuration for auctions run when [refreshing an ad slot](https://developers.google.com/publisher-tag/samples/refresh). To support auction configuration updates, later calls with the same `configKey` would replace the stored component `auctionConfig` for that `configKey` and slot, while `Slot.setConfig({ componentAuctions: [{configKey, null}] })` would remove it.

Component seller bids into the top-level auction will be interpreted by Ad Manager as “publisher payout net of fees or revshares” expressed in USD CPM units. Ad Manager will compare the bids of all sellers and the best contextual ad (which is selected via [dynamic allocation](https://support.google.com/admanager/answer/3721872?hl=en)), and will serve the ad with the highest bid.

## Next steps
Experimental support in GPT is now available. Please see the [GPT developer documentation](https://developers.google.com/publisher-tag/reference#googletag.config.componentauctionconfig) for further details. 

While we're not sure what support for multiple FLEDGE sellers might look like in Ad Manager long-term, we're committed to iterating toward a holistic solution based on what we learn from these tests and discussions with partners as we work toward a more privacy safe web. 

## Demo
A walkthrough of a basic setup for testing FLEDGE component auctions with Ad Manager via GPT is available [here](demo.md).
