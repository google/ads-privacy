# Dovekey

*Teams from across Google, including Ads teams, are actively engaged in industry dialog about new technologies that can ensure a healthy ecosystem and preserve core business models. Online discussions (e.g on GitHub) of technology proposals should not be interpreted as commitments about Google ads products.*

## Introduction
 
DOVEKEY simplifies the bidding and auction aspects of [TurtleDOVE](https://github.com/WICG/turtledove) by introducing a third-party KEY-value server. It is a modification of [SPARROW](https://github.com/WICG/sparrow) where the Gatekeeper acts as a simple key-value (KV) server. The scope of this proposal includes bidding and auction up until creative rendering. Reporting is strictly out-of-scope. 
 
The SPARROW proposal enhances TURTLEDOVE by introducing a trusted third party server (the Gatekeeper). This trusted server hosts bidding models and executes auction logic. Relative to the in-browser solution proposed by TURTLEDOVE, Sparrow provides more flexibility with bidding and consumes fewer browser resources.
 
With all its merits, we notice a few hurdles that may impede adoption by both ad tech and browser communities. First, SPARROW requires DSPs and SSPs re-implement bidding models and auction logic in the Gatekeeper. This is a significant undertaking which requires full trust in the Gatekeeper to safeguard proprietary technology and confidential information. Further, SPARROW offers limited means of establishing trust or guaranteeing user privacy.
 
Dovekey supports many of the same use cases as Sparrow while also mitigating these challenges:
1. It alleviates the need for the ad tech industry to re-implement logic on an external server, the KV server will cache the results of existing control and bidding logic.
2. It allows trustworthiness to be established via an open source auditing process, the  KV server will only support limited and well-defined functionality.
3. It can guarantee user privacy, the KV server can be implemented in a private or semi-private manner.
 
**The key idea of Dovekey is that we can get most of the benefit of SPARROW bidding even if the Gatekeeper server just acts as a simple lookup table** - a trusted Key-Value (KV) server which receives a Key (a contextual signal plus an interest group) and returns a Value (a bid).
 
## Proposed Workflow
 
### Browser requests interest group bids from KV server
 
Our proposed workflow is depicted in the following diagram. This workflow is meant to illustrate how a KV server can be used and not to define the final implementation.
 
<img src="https://user-images.githubusercontent.com/71041650/93560109-33278680-f936-11ea-92e5-3942b12711f3.png" width="640">
 
#### Explanation
1. The browser sends a regular contextual ad request as described in TURTLEDOVE.
2. The SSP returns the winning contextual ads, along with derived contextual signals (from SSP and DSP separately.) The final contextual signal is formed as the concatenation of those from a DSP and an SSP. [<sup>[1]</sup>](#alt1)
3. The browser constructs the key by combining the contextual signal and the interest-group ID (ig_id), and sends those keys in a request to a KV server. [<sup>[2]</sup>](#alt2)
4. The KV server returns all the bids associated with the requested keys. (*)
5. The browser runs a simple auction between interest-group candidates fetched from the KV server and the winning contextual ad. (*) [<sup>[3]</sup>](#alt3)
    * If a contextual ad wins, then the browser can proceed with rendering the ads.
    * If interest-group ad wins, the browser can send another request to the KV server to get the creative or creatives can be pre-fetched ahead of time via the interest group request, as discussed in the TURTLEDOVE proposal.
  
(*) Steps 4 and 5 are needed assuming the KV server can only perform lookup. It may also be possible for the KV server to run the final auction, in which case only the winning bids and creatives need to be returned.
 
### Ad tech providers populate interest group bids into KV server
 
DSPs can collaborate with SSPs to pre-populate the KV server with keys that maximize coverage and values that correspond to interest group-based bids.
 
From historical ad requests and tagging signals, along with information about previously missing keys that would need to be provided by the [aggregate reporting API](https://github.com/csharrison/aggregate-reporting-api), the DSP can make an informed decision to predict which keys (contextual signals + ig_id) will provide maximum coverage. Once a key is identified, the DSP can compute the respective values, i.e., creative_id and bid, and then pass this key-value pair, along with the contextual ad request, to the SSP.
 
Given a contextual ad request, creative_id, and the bid, the SSP can apply publisher brand safety & ad quality checks, pricing rules to calculate publisher payout. This publisher payout will be included in the final value, to be uploaded to the KV server. From the contextual ad request, the SSP constructs its contextual key in a way that all requests with the same key have the same brand safety checks and pricing rules.  The SSP then concatenates its key with the one constructed by the DSP thereby generating a final key for use on the KV server.
 
This process can be done either in batch (e.g. using past contextual ad requests) or online when the contextual ad request is made. Each SSP can independently decide which approach to take. The decision need not impact the browser implementation.
 
#### Namespaces and access controls
 
In order to protect the interests of advertisers and ad tech companies, only parties (e.g. DSPs, ad networks) specified as readers of an interest group should be able to submit bids for that interest group, and they can do so even if their supply partners (e.g. SSP) are not readers of the interest group. To address this requirement, the KV server can validate the DSP (domain) that submits each interest group bid and store the origin of the bid as part of the key or value. The browser may then perform access control checks to ensure interest-based bids it receives from the KV server are associated with the domain of an eligible DSP – a reader of an interest group.
 
## Sample Use Cases
 
Here we demonstrate several use cases, and how they can be supported by Dovekey.
* Budget management
  * The DSP remembers when it updated bids cached inside the KV server, and hence can decide when to refresh the cached bids. It should also be possible for the KV server to honor the TTL of each cached bid specified by the DSP.
  * If we have asynchronous “interest-group request” as in TURTLEDOVE, the DSP can send a signal to the browser that an interest group is out of budget, and hence the browser would not send the key to the KV server.
* Incorporating Browser-side Signals
  * DSP-provided Javascript code may instruct the browser to add more information about the interest group (e.g., recency) to the key.
* New Inventory
  * The DSP may have broad knowledge about the publisher and their inventory from the bid request stream, so it can choose to use a coarser contextual DSP-specific key part (e.g., domain instead of URL) to explore new inventory. DSP can then submit IG bids for such coarser keys to SSP to populate bids in the KV server.
* A/B Testing
  * DSP can append an experiment_id to the key as desired.
* Fraud and security
  * The bid is expected to be non-modifiable in the browser, and hence can be digitally signed by the DSP and/or SSP.
* Product-level Ads
  * We can apply the same mechanism as proposed in [Product-level Turtledove](https://github.com/jonasz/product_level_turtledove), by using the product ad components (web bundles) fetched ahead of time.

## Key-Value Server Options

### Trusted Server

This is the easiest to implement. The KV server will simply return the value for the requested key. This level of trust aligns with The Gatekeeper in the SPARROW proposal. Being simple allows the server to be easily auditable and reduces technical complexity. On the other hand, legal contracts, audits, and trust in the operator are the only privacy guarantees.

### Secure 2-Party Computation (2PC) System

This is a more challenging implementation with stronger privacy guarantees. Under the honest-but-curious model, the Secure 2PC system relies on crypto protocols and/or differential privacy to prevent a potentially compromised server from associating Interest Group IDs with a particular contextual ad request.

Secure 2PC may alleviate some of the computational challenges of a single-server PIR (mentioned below) and hence may better satisfy monetization efficiency, scalability, latency, bandwidth and infrastructure cost requirements, while providing stronger data protections. 

### Single-Server Private Information Retrieval

This is the most challenging implementation with the strongest privacy guarantees. A single server Private Information Retrieval (PIR) System ([Wikipedia](https://en.wikipedia.org/wiki/Private_information_retrieval)) ensures the server does not learn anything about the user data it receives. This opens the door to a non-trusted party running the server such as an SSP. On the other hand, many challenges exist to this type of implementation including scalability/cost and tight coupling with request construction in the browser.

## Appendix

### Alternative workflows and examples

<sup id=alt1>[1]</sup> **Step 2 [↩](#explanation)**
* An example of a contextual signal is the web page URL (plus a few coarse-grained dimensions like country, language, slot-size). DSPs and SSPs have the flexibility to specify their own contextual signals to maximize the coverage, while trading off with the bid accuracy. For example, it can be as fine-grained as a URL, or as coarse as a contextual category derived from a web page.
* If waiting for contextual request to finish incurs too much latency, we can alternatively construct the contextual signals in the browser by running a side-effect free Javascript function provided by an SSP (and possibly DSP, associated with each interest group in advance) in a browser-provided sandbox without network access. This way, the Step (3) can be executed in parallel with Step (1)

<sup id=alt2>[2]</sup> **Step 3 [↩](#explanation)**
* As an alternative to ig_id, each interest group can contribute its own part of the key using a custom JavaScript function provided by a DSP at the time such an interest group is joined by a browser. Such function would be run by browsers in a sandboxed environment and could be used to construct the key from the custom information about the interest group (e.g., recency).

<sup id=alt3>[3]</sup> **Step 5 [↩](#explanation)**
* One major difference between Dovekey and SPARROW is that the final auction happens in the browser. To ensure that the bid is trusted, we can, for example, require the bid to be non-modifiable in the browser, and protected with digital signatures by the SSP (and, optionally, DSP).

## Related Proposals
* [TURTLEDOVE](https://github.com/WICG/turtledove)
* [SPARROW](https://github.com/WICG/sparrow)
