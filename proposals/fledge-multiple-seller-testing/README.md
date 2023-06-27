# Ad Manager’s Testing of the Protected Audience API With Multiple Sellers
## Goal
We would like to outline how Ad Manager will support testing of the [Protected Audience API](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) with multiple sellers (i.e. exchanges) that might be working with Ad Manager publishers.

## Context 
To enable Protected Audience API testing for different exchanges on Ad Manager inventory, Ad Manager offers a lightweight API for exchanges to specify their auction configurations via the Google Publisher Tag (GPT) API. 

## Technical details
To enable different exchanges to test the Protected Audience API with publishers that use Ad Manager as an ad server, Ad Manager has added an experimental method to the GPT API, `Slot.setConfig({ componentAuctions: [{configKey, auctionConfig}] })`. Other sellers (i.e. exchanges) that work with a publisher can call this method before the publisher initiates an ad request to Ad Manager. GPT would collect all provided auction configurations, use those to populate the `componentAuctions` field of the top-level `auctionConfig`, and commence the top-level auction if the ad request is enabled for Protected Audience auctions.

Sellers might need to update their previously provided auction configuration for auctions run when [refreshing an ad slot](https://developers.google.com/publisher-tag/samples/refresh). To support auction configuration updates, later calls with the same `configKey` would replace the stored component `auctionConfig` for that `configKey` and slot, while `Slot.setConfig({ componentAuctions: [{configKey, null}] })` would remove it.

Component seller bids into the top-level auction will be interpreted by Ad Manager as “publisher payout net of fees or revshares” expressed in USD CPM units. Ad Manager will compare the winning bid of each component auction, including Ad Manager's own component auction for interest group bids of its buyers, as well as the best contextual ad (which is selected via [dynamic allocation](https://support.google.com/admanager/answer/3721872?hl=en)), and will serve the ad with the highest bid.  This comparison will be done through a [first price auction](https://blog.google/products/admanager/update-first-price-auctions-google-ad-manager/) - where Ad Manager will not share the bid of any auction participant with any other auction participant prior to completion of the auction.

## Next steps
Support in GPT is now available. Please see the [GPT developer documentation](https://developers.google.com/publisher-tag/reference#googletag.config.componentauctionconfig) for further details. 

We're committed to iterating toward a holistic solution based on what we learn from these tests and discussions with partners as we work toward a more privacy safe web. 

## Demo
A walkthrough of a basic setup for testing Protected Audience component auctions with Ad Manager via GPT is available [here](demo.md).
