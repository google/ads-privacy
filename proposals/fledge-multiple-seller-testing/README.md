# Ad Manager’s Testing of the Protected Audience API With Multiple Sellers
## Goal
We would like to outline how Ad Manager will support testing of the [Protected Audience API](https://github.com/WICG/turtledove/blob/main/FLEDGE.md) with multiple sellers (i.e. exchanges) that might be working with Ad Manager publishers.

## Context 
To enable Protected Audience API testing for different exchanges on Ad Manager inventory, Ad Manager offers a lightweight API for exchanges to specify their auction configurations via the Google Publisher Tag (GPT) API. 

## Technical details
To enable different exchanges to test the Protected Audience API with publishers that use Ad Manager as an ad server, Ad Manager has added an experimental method to the GPT API, `Slot.setConfig({ componentAuctions: [{configKey, auctionConfig}] })`. Other sellers (i.e. exchanges) that work with a publisher can call this method before the publisher initiates an ad request to Ad Manager. GPT would collect all provided auction configurations, use those to populate the `componentAuctions` field of the top-level `auctionConfig`, and commence the top-level auction if the ad request is enabled for Protected Audience auctions.

Sellers might need to update their previously provided auction configuration for auctions run when [refreshing an ad slot](https://developers.google.com/publisher-tag/samples/refresh). To support auction configuration updates, later calls with the same `configKey` would replace the stored component `auctionConfig` for that `configKey` and slot, while `Slot.setConfig({ componentAuctions: [{configKey, null}] })` would remove it.

Component seller bids into the top-level auction will be interpreted by Ad Manager as “publisher payout net of fees or revshares” expressed in USD CPM units. Ad Manager will compare the winning bid of each component auction, including Ad Manager's own component auction for interest group bids of its buyers, as well as the best contextual ad (which is selected via [dynamic allocation](https://support.google.com/admanager/answer/3721872?hl=en)), and will serve the ad with the highest bid.  This comparison will be done through a [first price auction](https://blog.google/products/admanager/update-first-price-auctions-google-ad-manager/) - where Ad Manager will not share the bid of any auction participant with any other auction participant prior to completion of the auction. For clarity, this means that neither the value of the best contextual ad nor the winning bid of any other SSP’s component auction is shared with any bidder in any other component auction - including Ad Manager’s own component auction. 

## Next steps
Support in GPT is now available. Please see the [GPT developer documentation](https://developers.google.com/publisher-tag/reference#googletag.config.componentauctionconfig) for further details. 

We're committed to iterating toward a holistic solution based on what we learn from these tests and discussions with partners as we work toward a more privacy safe web. 

## Demo
A walkthrough of a basic setup for testing Protected Audience component auctions with Ad Manager via GPT is available [here](demo.md).

## Manual testing
To help developers build, test, and debug Protected Audience API integrations, Ad Manager has developed a mechanism for developers to be able to ensure that the Google Publisher Tag (GPT) runs a Protected Audience auction when conducting local testing.  If you are interested, please sign up [here](https://services.google.com/fb/forms/uastringformultisellertestsignup/).

## FAQ
**Q: Does Ad Manager enforce a timeout for Protected Audience auctions?**
A: Ad Manager currently sets a timeout of 5 seconds for Protected Audience auctions, inclusive of all component auctions and the top level auction. Note that if the auction timeouts then the best contextual ad will be rendered, if there is one. 

We plan to run experiments with different timeouts on a small fraction of traffic to identify a timeout value that maximizes publisher performance and minimizes impact to user experience. We will continue to update this FAQ based on the results of those experiments.

**Q: How does Ad Manager decide which ad size to use for the PA auction?** 
A: Among all of the sizes a publisher specifies for a given ad slot, Ad Manager currently apply the following logic to select the size used for the PA auction:
- Ignore any placeholder sizes, where placeholder is defined as a square creative with a side of <= 5 pixels 
- Look if there are any sizes that are part of the set of supported ad sizes defined [here](https://support.google.com/admanager/answer/1100453?hl=en). If there are, choose the largest supported size by area (width * height)
  - For clarity, the set of supported ad sizes includes all of the ad sizes listed under  “Top-performing ad sizes”, “Other supported ad sizes”, and “Regional ad sizes”. 
- If not, choose the largest remaining size (i.e. that isn’t in the list of supported ad sizes) by area (width * height)

This logic is subject to change as we experiment with optimizations. We will continue to update this FAQ if the implementation changes. 

**Q: How does Ad Manager plan to use Chrome-facilitated testing labels?**  
A: Please see our November 2023 testing update [here](https://support.google.com/admanager/answer/13178817?hl=en&ref_topic=12264880&sjid=16287796969466812891-NA), which describes our plans in detail. 

**Q: What reporting is available to publishers in Ad Manager about Protected Audience auctions won by non-Google sellers?**
A: Ad Manager does not currently provide reporting to publishers for impressions won by non-Google component auctions, though we are working on providing such reporting in the future.  Publishers should be able to obtain that reporting directly from the non-Google seller who won the auction.

**Q: Does Ad Manager render ads in an iFrame or a Fenced Frame?**
A: Until Fenced Frames are required, Ad Manager is planning to render the majority of traffic using iFrames, but may run small scale experiments to evaluate Fenced Frames as the feature continues to be developed by Chrome. 

