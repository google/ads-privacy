# Dovekey Scalability

This document addresses the question of Dovekey scalability, outlining some ways to reduce key cardinality. These approaches appear feasible though they still need to be validated on real-world data. 

In order to retain coverage and functionality without inflating the number of keys, we introduce the concepts of a *coarse key* and a *specific key* for further discussion amongst the community. These concepts provide a means to balance precision and control with key cardinality.

* *Coarse keys*: a single *coarse key* can cover the needs of a group of publishers and advertisers. These keys are constructed from properties common to multiple publishers or advertisers.
  * Bidding (DSP) - in order to achieve reasonably accurate bids, it’s not necessary to have a separate key for every possible slice. A coarse level key should be able to capture the majority of the relevant signal. Features like bucketized bid (e.g., advertiser specified CPC bid), bucketized ad slot quality, and few in-browser signals (coarse granularity geo location, browser language, bucketized recency and frequency), should suffice. Ad slot quality can encode a publisher’s features useful for bid valuation such as page category, average conversion rate, ad slot size, above/below the fold, etc.
  * Advertiser controls (DSP) - each key should represent the specific control for a particular ad or interest group. That is, ads that impose the same control, e.g., sensitive category exclusions, can be grouped in the same key.
  * Publisher controls (SSP) - each key should represent the publisher's control. That is, publishers that impose the same control, e.g., sensitive category exclusions, can be grouped in the same key.
* *Specific keys*: Some publishers and advertisers impose fine grained controls, and hence require dedicated keys. These keys are constructed from properties specific to a publisher or advertiser at the cost of higher cardinality. Examples of *specific keys* include domain (or URL) and interest group IDs.  
  * Though in theory the *specific key* space of publisher x interest group ID may reach trillions, in practice it can be much smaller - a tradeoff between cardinality and precision. Our initial analysis indicates that this number can be well within a reasonable range.
  
Given a contextual ad request, an interest group can match one or more *coarse keys* and *specific keys*, which are associated with different bids. Dovekey could be extended to support multiple keys to be queried for the same ad impression opportunity and allow specifying the priority for different keys. Since the DSP constructs the keys to associate with a bid using server-side logic, it can also specify which keys and values to prioritize if multiple keys match.

## Implementation Options

### Meeting publisher and advertiser needs within the key space

By using a combination of *coarse keys* and *specific keys*, the DSP and SSP can cover the needs of both publishers and advertisers while also limiting key cardinality.

1. Individual publishers or advertisers who use sophisticated controls can be covered with *specific keys*, i.e., one key per unique publisher-advertiser slice. An example of a slice might include a publisher who defines specific controls for the homepage of a particular domain.
2. Sets of publishers and advertisers who use only generic controls (e.g., exclusion of sensitive categories) can be covered with a small number of *coarse keys*.
3. A hybrid of the options above - advertisers with simple controls (represented with a *coarse key*) can bid on publishers using sophisticated controls (represented with a *specific key*) via a combination of specific-SSP-key and coarse-DSP-key (and vice versa). This can be achieved as SSP and DSP construct their keys independently. 

### TURTLEDOVE for blocking controls, Dovekey for bidding

One reason for the potential high key cardinality in Dovekey is that the key-values need to capture the publisher and advertiser brand safety and other controls, which ad platforms are expected to enforce accurately. In Dovekey, this requires encoding such controls into dedicated *specific keys* for individual publishers and advertisers.

One way to alleviate this problem is to use the Dovekey framework for bidding, while implementing brand-safety mechanisms via in-browser logic as in the original TURTLEDOVE. In this variation, *specific keys* wouldn’t be necessary for publisher and advertiser controls, and can be added optionally to improve bid accuracy. With Dovekey, bid valuation can still happen server-side, allowing to keep in-browser logic simple and limited to filtering out ineligible bids.
