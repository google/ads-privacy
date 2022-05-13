# User Agent generalization in RTB bid requests

## Disclaimer

_Teams from across Google, including ads teams, are actively engaged in industry dialog about new technologies that can ensure a healthy ecosystem and preserve core business models. Online discussions (e.g on GitHub) of technology proposals should not be interpreted as commitments about Google products. This proposal is specific to real-time bidding and does not reflect plans of any browser, including Chrome, about similar or related changes._

## Background

The raw *User-Agent* string often found in RTB bid requests (`Device.ua` field in OpenRTB) typically serves for two main broad categories of use cases:

*   Device and browser type detection for use in campaign targeting and bidding optimization algorithms.
*   Detecting and filtering invalid traffic, for instance, by observing anomalies associated with specific user agents.

At the same time, the information about the user's device carried in the raw User-Agent string is fairly granular as it can include, for instance, minor and micro browser and OS versions, device firmware build and more.  Such granular information may create risks of covert tracking by bad actors, who might attempt to re-identify end user devices even when cookies or device advertising IDs are not available (e.g., in private browsing mode) or they are reset.

## Summary

Covert tracking risks can be reduced by _generalizing or redacting_ the information about the User-Agent in bid requests to achieve desired privacy goals (for example, some level of K-anonymity).  To improve privacy protections in real-time bidding, we propose approaches to generalizing the user agent information in bid requests, which enable the continued support of the device and browser type detection use case.  We plan to look at how pre-bid IVT detection and filtering use cases can be supported in a privacy-centric manner separately.

### Related work

Chrome [announced](https://groups.google.com/a/chromium.org/g/blink-dev/c/e3pZJu96g6c) [plans](https://blog.chromium.org/2021/09/user-agent-reduction-origin-trial-and-dates.html) to [reduce the User-Agent](https://blog.chromium.org/2021/05/update-on-user-agent-string-reduction.html), so that the reduced User-Agent string will continue to provide access to “_browser major version, platform name, and distinguish[ing] between desktop and mobile_”. 

The generalization techniques described below are largely aligned with the approach suggested by Google Chrome and other major browsers.  The major difference is the scope and consistency of privacy protections: an RTB exchange implementing User-Agent generalization can extend the privacy benefits across many device types, browser types and apps, including those that might not yet have taken similar steps to protect users against passive fingerprinting risks. In particular, many older devices that don't receive regular updates might never benefit from client-side privacy improvements. 

RTB exchanges could apply User-Agent generalization on inventory that the publishers and/or the users view as more privacy-sensitive (for example, where users opted out of personalized advertising) or across all inventory.

## User Agent generalization in bid requests

### Structured User Agent

[Structured User-Agent](https://github.com/google/ads-privacy/tree/master/experiments/structured-ua) (SUA) is a bid request representation of the User-Agent information that breaks it down into a strongly-typed object with fields describing the browser, platform, architecture and device.  The Structured User-Agent is available in the [Google Authorized Buyers protocol](https://developers.google.com/authorized-buyers/rtb/realtime-bidding-guide) (`BidRequest.user_agent_data`) and as an [OpenRTB extension](https://developers.google.com/authorized-buyers/rtb/openrtb-guide) (`BidRequest.device.sua`).  Consider this example of both representations of the User-Agent for the same request:

_Raw User-Agent string:_

```
Mozilla/5.0 (Linux; Android 11; M2007J20CG Build/RKQ1.200826.002; wv)
AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/96.0.4664.92
Mobile Safari/537.36 [MyApp:CustomSignals:ABC123]
```

_Structured User Agent object, with generalized versions (in OpenRTB/JSON):_

```
"browsers": [
  {"brand": "Mozilla", "version": ["5", "0"]},
  {"brand": "AppleWebKit", "version": ["537", "36"]},
  {"brand": "Version", "version": ["4", "0"]},
  {"brand": "Chrome", "version": ["96", "0", "0", "0"]},
  {"brand": "Mobile Safari", "version": ["537", "36"]}
],
"platform": {"brand": "Android", "version": ["11"]},
"mobile": 1,
"model": "M2007J20CG"
```

Notice that in the example above, some version information is generalized in the Structured User Agent object, specifically the Chrome version where minor version components are all replaced by zeros.

Version generalization can be applied to most browsers and platforms that appear in the SUA object, but there are exceptions.  The example's platform version doesn't need generalization because it contains only a major number (11), and Safari's _Version/4.0_ already has a zero in the minor version so generalization doesn't change it.  The full versions of the _AppleWebKit_ and _Mobile Safari_ browser entries, including a non-zero minor version, are still intact because _537.36_ is a frozen value, so it doesn't contribute any additional entropy.  These heuristics help preserve the details that have no impact on user privacy protections, seeking a balance with backwards compatibility.

After adopting the Structured User Agent, bidders should not need the raw User-Agent string in bid requests for the device and browser type detection and targeting, as they should be able to derive sufficient information from the SUA.  The RTB protocol also includes the Device object with details about the detected device (brand, screen size etc.), largely determined from the _User-Agent_ header.


### Generalized User Agent string

The presence of the traditional User-Agent string field in a bid request can be important for backwards compatibility, as bidders might rely on the information parsed from the UA string. That said, there might be no gain for privacy from the Structured User Agent representation if the high-entropy information is still available in the User-Agent string field.  That field should also be generalized to improve privacy protections.  Instead of carrying the original User-Agent header value, the `BidRequest.ua` field can be populated with a string that follows the expected User-Agent syntax but contains only the amount of information comparable to the information in the SUA.  Using the example above, the generalized string can look as follows:

_Generalized User Agent string:_

```
Mozilla/5.0 (Linux; Android 11; M2007J20CG; wv)
AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/96.0.0.0
Mobile Safari/537.36
```

The generalized string is smaller and simpler than the original but still carries the same essential data as the Structured User Agent: browser names and versions, platform name and version, and the device model.  It can omit details such as the device's build/firmware version and the mobile SDK, which are not represented in the Structured User Agent.  The two fields can carry the same information (and amount of entropy), only in different forms.  If the SUA representation only provided major versions, the same constraint would also apply to the generalized UA string.

Notice that in this generalized UA string it is particularly important to reproduce the behavior of each browser or platform when we minimize version strings.  For example, in the string above we have _Chrome/90.0.0.0_, not just _Chrome/90_.  That's what the original UA would contain if it was issued by Chrome's initial build for a new major version, and also by Chrome's own reduced User-Agents.  The objective is to be compatible with code that parses the string; for example, some parsers can be detecting Chrome with a regular expression that expects exactly 4 numeric version components.  In the Structured User Agent object this backwards-compatibility concern doesn't exist, but even in that representation, the minimized version components should also be replaced by zeros for consistency.

Generalized UA strings should be compatible with the existing User-Agent parsers and classifiers that work with the original values; in other words, a generalized User Agent string should yield the same or a very similar classification as the original one (subject to the expected loss of detailed information, for instance, about minor and micro versions). The User-Agent string generalization algorithm needs to be evaluated for backwards compatibility – for instance, by testing raw and generalized strings with popular classification libraries.

### UA string generalization examples

<table>
  <tr>
   <td>Original
   </td>
   <td>Generalized
   </td>
  </tr>
  <tr>
   <td>Mozilla/5.0 (iPad; CPU OS 10_3_4 like Mac OS X) AppleWebKit/603.3.8 (KHTML, like Gecko) Mobile/14G61
   </td>
   <td>Mozilla/5.0 (iPad; CPU OS 10_<strong>0</strong> like Mac OS X) AppleWebKit/603.<strong>0.0</strong> (KHTML, like Gecko) Mobile/14G61
   </td>
  </tr>
  <tr>
   <td>Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.45 Safari/537.36
   </td>
   <td>Mozilla/5.0 (Windows NT 6.<strong>0</strong>; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.<strong>0.0.0</strong> Safari/537.36
   </td>
  </tr>
  <tr>
   <td>Mozilla/5.0 (Linux; Android 11; M2007J20CG <del>Build/RKQ1.200826.002</del>; wv) <br/>
       AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0 Chrome/96.0.4664.92 Mobile Safari/537.36
       <del>[MyApp:CustomSignals:ABC123]</del>
   </td>
   <td>Mozilla/5.0 (Linux; Android 11; M2007J20CG; wv) AppleWebKit/537.36 (KHTML, like Gecko) Version/4.0
       Chrome/96.<strong>0.0.0</strong> Mobile Safari/537.36
   </td>
  </tr>
</table>

The table above illustrates a few examples.  In the first row, an old iOS device uses a non-frozen `AppleWebKit` version _603.3.8_, so in that case the version is generalized to _603.0.0_.  Notice also that iOS _10_3_4_ is not generalized to _10_0_0_ but _10_0_ because that's how iOS represents a major version before any patches.  In the second row, we generalized Windows from _6.3_ to _6.0_; because Microsoft used to update only the minor version for older Windows releases, that reduces "Windows 8.1" to "Windows Vista or better".  Modern versions of Windows don't have that problem anymore, so another special case for Windows versions is not necessary.  Finally, in the third row we have a longer mobile app User-Agent that shows two significant generalizations: removal of the device's build version and an app-specific suffix.

Note that some elements that may be redacted from the generalized User-Agent string can still be made available via other, existing fields in the RTB protocol (e.g., OpenRTB), in particular mobile app ID and name as well as mobile SDK.  Bidders should prefer these dedicated fields, which offer a higher level of abstraction and are easier to use, instead of parsing the properties found in the raw User-Agent string. 

## Bid request changes

A new field can be added to the RTB protocol to communicate what privacy protections are applied to the bid request fields.  This field will be initially an OpenRTB extension: `BidRequest.ext.privacy_treatments`.  Its type is an `Object PrivacyTreatments` with the following fields: 

<table>
  <tr>
   <td style="background-color: #893d36">Attribute
   </td>
   <td style="background-color: #893d36">Type
   </td>
   <td style="background-color: #893d36">Description
   </td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">ua
   </td>
   <td style="background-color: #dde5f0">integer
   </td>
   <td style="background-color: #dde5f0">Privacy treatment for the User-Agent string.  See Table 1.
   </td>
  </tr>
  <tr>
   <td style="background-color: #eef2f8">sua
   </td>
   <td style="background-color: #eef2f8">integer
   </td>
   <td style="background-color: #eef2f8">Privacy treatment for the Structured UserAgent.  See Table 2.
   </td>
  </tr>
</table>

Table 1: Privacy treatments for User-Agent

<table>
  <tr>
   <td style="background-color: #893d36">Name
   </td>
   <td style="background-color: #893d36">Value
   </td>
   <td style="background-color: #893d36">Description
   </td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">USER_AGENT_FULL
   </td>
   <td style="background-color: #dde5f0">0
   </td>
   <td style="background-color: #dde5f0">No generalization / full value (the default).
   </td>
  </tr>
  <tr>
   <td style="background-color: #eef2f8">USER_AGENT_COARSENED
   </td>
   <td style="background-color: #eef2f8">1
   </td>
   <td style="background-color: #eef2f8">Value may be generalized (minimized).
   </td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">USER_AGENT_REDACTED
   </td>
   <td style="background-color: #dde5f0">2
   </td>
   <td style="background-color: #dde5f0">Value is redacted.
   </td>
  </tr>
</table>

Table 2: Privacy treatments for Structured User Agent

<table>
  <tr>
   <td style="background-color: #893d36">Name
   </td>
   <td style="background-color: #893d36">Value
   </td>
   <td style="background-color: #893d36">Description
   </td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">USER_AGENT_DATA_FULL
   </td>
   <td style="background-color: #dde5f0">0
   </td>
   <td style="background-color: #dde5f0">No generalization / full value (the default).
   </td>
  </tr>
  <tr>
   <td style="background-color: #eef2f8">USER_AGENT_DATA_COARSENED
   </td>
   <td style="background-color: #eef2f8">1
   </td>
   <td style="background-color: #eef2f8">Value may be generalized (minimized).
   </td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">USER_AGENT_DATA_REDACTED
   </td>
   <td style="background-color: #dde5f0">2
   </td>
   <td style="background-color: #dde5f0">Value is redacted.
   </td>
  </tr>
</table>

Example (OpenRTB/JSON) for a request where the `BidRequest.ua` value is coarsened:

```
"privacy_treatments": { "ua": 1 }
```

These fields can provide transparency into the bid request field generalization that an exchange may choose to apply to a given request, allowing bidders to anticipate and act on these changes, and make it easier to evaluate their impact.

## Bidder changes

Bidders should not need to make any changes to their User-Agent parsing code to adapt to these generalized User-Agent strings.  If the `BidRequest.ua` field is used, it should continue to be parsed to extract the datum of interest as usual.  An exchange applying the generalization  might not document specific implementation in detail; all examples provided  are only for illustrative purposes.  The User-Agent generalization implementation may evolve over time to improve privacy protections and backwards compatibility with the existing bidder User-Agent string classification by bidders for the majority of traffic.  However, some high-level principles should apply:

* The generalized string will follow the same general syntax / structure as the original.
* The only potential changes compared to the original string are zeroing of version numbers and omission of certain high-entropy values.  The exchange should not replace any values with semantically different ones, e.g. normalizing all desktop platforms to a single platform identifier.
* The full list of browser entries shall be preserved, at least for the well-known entries (actual browser or engine names), as well as core platform and device identification.

To recap, the Structured User Agent is the ideal source of information for targeting by device, browser, and platform.  The generalized User Agent string is a transitional, backwards-compatible approach that allows the exchange to improve user privacy protections without requiring bidders to immediately migrate away from the User-Agent string to the Structured User Agent.  Both fields can be generalized in a way that minimizes device-identifying entropy while preserving utility and compatibility.
