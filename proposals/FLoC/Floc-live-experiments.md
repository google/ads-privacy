# Methodology on running FLOC simulations on live traffic and evaluating advertiser metrics

## Goal

To simulate FLoCs as described in the [whitepaper](https://github.com/google/ads-privacy/blob/master/proposals/FLoC/FLOC-Whitepaper-Google.pdf) 
based on a small percentage of live traffic ads network data and [Chrome’s FLOC explainer](https://github.com/WICG/floc).

## Methodology

Google Ads continuously runs a/b experiments to improve the accuracy and performance of our advertising systems. 
The FLoC results we [posted](https://blog.google/products/ads-commerce/2021-01-privacy-sandbox/), 
comes from one of these live traffic experiments. 
The split works like the following: for a small but statistically significant portion of live traffic, 
inbound ad requests from publishers are diverted into treatment and control arms.
1. For the control arm, our targeting systems used traditional 3P cookies 
2. For the treatment arm, the 3P cookie signal was redacted from the input to our interest-based audience targeting models 
and a simulated FLoC using 
[Simhash Sorting-LSH](https://github.com/google/ads-privacy/raw/master/proposals/FLoC/FLOC-Whitepaper-Google.pdf) 
technique was added. 
The rest of our systems were left unchanged - meaning we honor the privacy settings included in the 3P cookie and models 
for our other targeting types continue to have access to these cookies (eg. Remarketing). 
Note: remarketing is considered an orthogonal problem as we expect it to be supported by a 
[different API](https://github.com/WICG/turtledove/blob/master/FLEDGE.md).

The experiment results were compared to the control for interest-based taxonomic
[Google Audiences](https://support.google.com/google-ads/answer/2497941) 
such as In-market 
(e.g. people interested in purchasing backpacks) and Affinity (e.g. sports fans.) 
Then a variety of metrics were evaluated to understand impact to accuracy and reach. 
Some such metrics include precision/ recall (of our audience models), and advertiser metrics such as advertiser efficiency 
(number of conversions & conversions per dollar). The results were analyzed for statistical relevance as is typical in an a/b test setup. 
The treatment arm reported 95% conversion per dollar (CPD) value for the advertisers as compared to the control arm. 

This methodology can be followed by others to experiment with how well FLOC might perform within their own advertising systems.

## Next Steps

As a next step, we plan to participate in the [Chrome origin trial live FLoC ids](https://developer.chrome.com/blog/privacy-sandbox-update-2021-jan/)
(when available) and begin testing directly 
with advertisers within Google Ads, in Q2.

