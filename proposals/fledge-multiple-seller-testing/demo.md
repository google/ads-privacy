# Demo for testing FLEDGE component auctions with Ad Manager

This document provides a walkthrough of a basic integration with GPT’s ComponentAuction API to test FLEDGE component auction experimental support in Ad Manager, explains how the demo code works, and shows how to use Chrome DevTools for FLEDGE debugging. Most of the same information can be found on the demo site too.

The demo is available at: https://fledge-multi-seller-with-gpt.glitch.me. You can view the source on glitch or remix to create your own version of the demo. The demo simulates the setup of a non-Google seller and its buyer participating in a FLEDGE component auction via the experimental GPT API.
 
First, follow the instructions at https://developer.chrome.com/blog/fledge-api/#flags to start Chrome with the FLEDGE API and related features enabled. There are two stages to this demo:

- First, you visit the advertiser site in the demo to join an interest group. This step is no different from the original FLEDGE demo. You should observe the ‘join’ event on the DevTool->Application->InterestGroup tab for an interest group owned by ‘"https://fledge-multi-seller-with-gpt.glitch.me". 
(Note that, to simplify the demo, we are making the component seller, the buyer and the publisher share the same "https://fledge-multi-seller-with-gpt.glitch.me" origin)

- Next, you visit the publisher site which loads GPT and runs a script that creates an AuctionConfig for the component seller "https://fledge-multi-seller-with-gpt.glitch.me" and passes it to the GPT API to participate in the component auction.

```javascript
const auctionConfig = {
  seller: "https://fledge-multi-seller-with-gpt.glitch.me", // should https & same as decisionLogicUrl's origin
  // x-allow-fledge: true
  decisionLogicUrl: "https://fledge-multi-seller-with-gpt.glitch.me/ssp/decision-logic.js",
  interestGroupBuyers: [
    "https://fledge-multi-seller-with-gpt.glitch.me",
  ],
  auctionSignals: { auction_signals: "auction_signals" },
  sellerSignals: { seller_signals: "seller_signals" },
  perBuyerSignals: {
    "https://fledge-multi-seller-with-gpt.glitch.me": {
      per_buyer_signals: "per_buyer_signals",
    }
  }
};

auctionSlot.setConfig({
  componentAuction: [
    {
      configKey: "https://fledge-multi-seller-with-gpt.glitch.me",
      auctionConfig: auctionConfig,
    }
  ]
});
```

Ad Manager’s GPT tag will trigger the call to `navigator.runAdAuction()` to run the on-device auction to select an ad.

Both component-level auctions and the top-level auction will be configured and run, and the ad in the interest group previously joined by the browser should always win and be rendered in an `<iframe src=urn:uuid>`. (an unrealistically high bid is submitted in the generateBid function to beat any highest-ranked contextual bid)

In this demo, we use a test publisher account and a test ad unit that are configured to always initiate  FLEDGE auctions. 

There are a few ways to verify the FLEDGE auction happen as expected:

1. Observe the ad in the interest group joined in the first step of the demo to be rendered.
2. On the `DevTool->Application->Interest Group` tab, expect to see the ‘bid’ and ‘win’ events from the  "https://fledge-multi-seller-with-gpt.glitch.me" interest group joined previously in this demo.
3. In the console log, expect to see signals logged from the auction/reporting worklet. In this demo, we log rich signals from scoreAd(), reportResult(), generateBid(), reportWin().
4. Record a trace on the DevTool->Performance tab, expect to see the activities on the Auction/Bidding worklets.

Please raise an issue or leave a comment if the above flow does not work as described.

