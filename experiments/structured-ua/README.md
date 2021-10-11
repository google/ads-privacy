# Experiment proposal: Structured User Agent

**NOTE: See proposal for [OpenRTB Structured UserAgent Specification](https://github.com/google/ads-privacy/blob/master/experiments/structured-ua/openrtb.md).**

*This experiment proposal does not reflect plans of any browser, including Chrome, to support similar means of in-browser ad personalization. While publicly sharing the results of these experiments can lead to productive discussions between ad tech companies as well as browser vendors and privacy advocates on how ad personalization can be supported with stronger user privacy protections, these discussions may not necessarily lead to browsers building support for the use case or the flow similar to the one proposed in this experiment.*

*The experiment may influence, but does not necessarily represent Google’s current or future real-time bidding product plans.*

## Background

The `User-Agent` header, which identifies web browsers (or equivalent client software), allows web servers to customize content for compatibility with the client's browser, operating system and platform, or supported extensions. The header has found usage in other pieces of web infrastructure – web crawlers, CDNs – and in advertising networks. In RTB protocols, the `User-Agent` shared via bid requests supports targeting by important criteria including device type, browser, operating system or mobile vs. desktop environment, as well as detecting invalid traffic. However, industry support for this header is about to change, primarily due to concerns about user privacy. This proposal is a first step to evolve the RTB protocol to preserve the critical use cases of User Agent information, in a way that's compatible with other initiatives to improve privacy protections.

Most of the information in the `User-Agent` is obsolete for its intended purposes: when it comes to compatibility, this header enabled "browser sniffing" techniques necessary in the early years of the web with weak HTML standards, competing browsers that could be incompatible in basic features, and popular native plugins. These problems are now largely in the past, and even sites that support bleeding-edge features will typically do feature detection instead of sniffing.

One of the legacy of those early years of the Web is that the `User-Agent` accumulated a large variety of information. Typical UA strings will give away a browser's full version and build number, the hardware platform it's running on, the HTML rendering engine, and a few "lies" such as `Mozilla/5.0`, masquerading as a browser that's almost 20 years old today. These bits of information create a significant amount of entropy, so much that the number of unique UA strings in use runs in the hundreds of millions. This makes the `User-Agent` string a potential fingerprinting vector, as demonstrated by [amiunique.org](https://amiunique.org).

The proposal that follows doesn't break new ground; it explores updates to the RTB protocol to align with a potential future for the web, where the traditional `User-Agent` header may be unavailable or frozen and thus providing little value. This experiment will be focused on the usage of User Agent information for targeting; other uses of this information, such as invalid traffic detection, will be out of scope of the current proposal.

### Existing work

Browser industry's privacy-motivated effort to evolve the functionality provided by the `User-Agent` header is already in course. Major browser vendors discussed potential plans to "reduce" this header: limit the number of fields and their granularity to a few elements, e.g. major browser version, or a few values, e.g. one for desktop and one for mobile). Meanwhile, the [User Agent Client Hints](https://wicg.github.io/ua-client-hints/) proposal, if/when supported by browsers, would implement a new set of headers (`Sec-CH-UA-*`) and Javascript APIs that would offer most of the same information, but with stronger privacy constraints and controls.

## Proposal

This experiment adds a *Structured User Agent* object to the RTB protocol, for example (in OpenRTB/JSON syntax):

```json
"useragent": {
 "browser": [{"brand": "Chrome", "version": ["80", "0"]}],
 "platform": [{"brand": "Windows", "version": ["10"]}],
 "architecture": "ARM64",
 "model": "Pixel 4",
}
```

This object will contain information sent by either the traditional `User-Agent` or the new proposed `Sec-CH-UA` headers. But the object is modeled after the latter, so that it's capable of carrying all data from the new headers, but not necessarily all data that can be extracted from the old header. This structure won’t provide some existing fields, such as identifiers of rendering engines or browser plugins. Here's how the new *Structured User Agent* can be populated based on each header type:

- If a request to an advertising exchange only contains the old `User-Agent`, it can be parsed (which exchanges typically already do for other reasons), extracting fine-grained fields such as browser brand or device model, to populate the corresponding fields in the structured useragent object in the Bid Request. Information that's produced by the UA string parser but isn't supported by the *Structured User Agent* object will be ignored.

	- Non-browser environments like mobile app SDKs typically rely on the `User-Agent` to carry other information, e.g. for carrier and network. These can be supported by other existing fields in the bid request (`Mobile`/`Device` objects). If necessary we can iterate on the experiment by adding more fields, but more likely in those objects rather than the *Structured User Agent*.

- If a request to an advertising exchange contains the new `Sec-CH-UA-*` headers, their values will be used. If both kinds of headers are available, `Sec-CH-UA-*` headers will be preferred. In this case the mapping is more straightforward, for example the main `Sec-CH-UA` header contains only two subfields (browser brand and versions) which can be easily separated to populate the `browser.brand` and `browser.version` fields. However, exchanges may normalize the values (if necessary) for consistency with the old header, to provide more accurate browser or device type classification, and/or to further protect user privacy

The RTB protocol's traditional `User-Agent` field could be removed from bid requests as part of this experiment. Long-term, the RTB industry should explore reducing the reliance on the traditional `User-Agent` header, finding alternatives for each of its uses. The *Structured User Agent* can be an appropriate solution for the device and browser type targeting use-case. 

### Privacy benefits

The new, structured field is also much friendlier to other privacy-enhancing techniques such as enforcing k-anonymity:

- Even if this field is fully provided (all subfields populated if the corresponding data is available), the result will contain less entropy than the old `User-Agent` field. The structured field doesn't support information such as rendering engine name and version, compatibility appendages such as "like Gecko", etc.

- If we still need to reduce entropy to prevent re-identifiability of individual users, the structured field makes it easier to generalize user agent information by removing only specific subfields, while continuing to support coarse-grained targeting. In the example above, we could decide to keep the browser's brand but omit its version, and completely omit the platform and architecture but still deliver the device model, etc. These fine-grained redaction decisions can be based on data about the contribution of each subfield to both re-identifiability risks and targeting (but this is not in scope of the current experiment).

This proposal is complementary to the [Privacy Sandbox](https://www.chromium.org/Home/chromium-privacy/privacy-sandbox) initiatives, in particular, [Privacy Budget](https://github.com/bslassey/privacy-budget) and [User-Agent string](https://github.com/WICG/ua-client-hints) proposals. The latter explores partial redaction of details in the `User-Agent` header, its freezing, and the transition to the new `Sec-CH-UA` headers. The current proposal carries the same ideas on the RTB protocol. However, RTB industry participants don't need to wait for the full transition of browsers (and other infrastructure such as ad tags) to experiment with and apply these ideas, since the structured field can be populated based on the `User-Agent` header as long as that's the status quo.

The `User-Agent` header is also used by mobile app SDKs, and at this time it's not clear whether or when these will make any changes to the header. Current discussions on freezing the `User-Agent` header are specific to certain browsers, including mobile browsers but not including app SDKs; and the CH-UA specification makes no mention of mobile SDKs and doesn't contain any headers/fields exclusive to that usage. But mobile SDKs make other signals available for device and model identification, and the RTB protocols have multiple fields in other places (`Mobile`/`Device` objects) for those signals such as screen size, device type, carrier, etc.

### Success metrics

In the experiment, exchanges can populate the new, structured field in the bid requests, leaving the old User Agent string field empty. The experiment can be restricted to a small fraction of inventory to limit any potential impact, and bidders that participate in the experiment should not be meaningfully disadvantaged in the auction. Bidders should implement support to process the new field in requests that carry the *Structured User Agent*.

This experiment can be evaluated via two dimensions:

1. Functionality: we expect the new field to support the same targeting functionality, so bidders who build support for the new field should not experience noticeable loss of targetable inventory, impressions or revenue.

2. Privacy: comparing the entropy of the new field to the old and estimating k-anonymity of the user agent data shared via bid requests in both cases.

The criteria of success is avoiding impact on targeting functionality while maximizing positive changes to privacy protections (significantly less entropy than the old field). Bidders should maintain the same ability for targeting by browser or device type.

### Limitations

The full `User-Agent` header has other legitimate uses in RTB than just targeting; in particular IVT (invalid traffic) detection typically uses that header for detecting non-human traffic and anomalies. These usages should be substituted by alternative solutions that would not require passing high-entropy fields in bid requests, for instance, based on the ideas discussed in the [Trust Tokens](https://github.com/WICG/trust-token-api) proposal. We intend to explore how IVT detection could be supported in the RTB protocol while strengthening privacy protections in a separate proposal.

