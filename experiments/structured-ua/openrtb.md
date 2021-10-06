## Structured UserAgent Specification

(Proposal for adding into OpenRTB 2.x specification, initially as a Standard Extension)

### Background

This document proposes a standard OpenRTB specification for the Structured
UserAgent, a new signal previously launched by Google as part of
[Google's RTB protocol](https://developers.google.com/authorized-buyers/rtb/downloads/realtime-bidding-proto),
also supported by Google as an
[OpenRTB extension](https://developers.google.com/authorized-buyers/rtb/downloads/openrtb-adx-proto).
See also Google's early
[public proposal](https://github.com/google/ads-privacy/blob/master/experiments/structured-ua/README.md).

*Commentaries that are not intended to be part of the specification are in this
style.*

### 3.2.18 Object: Device

*Update the specification for the following fields:*

<table>
  <tr>
   <td style="background-color: #893d36">Attribute</td>
   <td style="background-color: #893d36">Type</td>
   <td style="background-color: #893d36">Description</td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">ua</td>
   <td style="background-color: #dde5f0">string; recommended</td>
   <td style="background-color: #dde5f0">User agent string.</td>
  </tr>
  <tr>
   <td style="background-color: #eef2f8">sua</td>
   <td style="background-color: #eef2f8">object; recommended</td>
   <td style="background-color: #eef2f8">
     Structured user agent defined by a UserAgent object (Section 3.2.<em>NEW1</em>).</td>
  </tr>
</table>

### 3.2._NEW1_ Object: UserAgent

This object contains information about the user agent, populated from either the *User-Agent* header or from the *Sec-CH-UA* headers (https://wicg.github.io/ua-client-hints/), collectively referred to here as "UA headers".
The *User-Agent* header contains information about the device's browser and platform, including some data that may be used to populate other objects such as `Device` and `App`.
The `UserAgent` object is not meant to replace or duplicate other high-level objects, but it should provide a structured, well-typed alternative to the `Device.ua` field.
It's also intended as an abstraction over all UA headers present in a specific request.

The `UserAgent` object is also designed to be simple and to clean up the long legacy of the *User-Agent* header which often contains redundant or obsolete information and requires complex parsing and classification effort.
`UserAgent` only contains fields with essential information that's found in a majority of UA headers, especially browser and platform identification and versions.

<table>
  <tr>
   <td style="background-color: #893d36">Attribute</td>
   <td style="background-color: #893d36">Type</td>
   <td style="background-color: #893d36">Description</td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">browsers</td>
   <td style="background-color: #dde5f0">array of object; recommended</td>
   <td style="background-color: #dde5f0">
     Each BrandVersion object (see Section 3.2.<em>NEW2</em>) identifies a browser or similar software component.</td>
  </tr>
  <tr>
   <td style="background-color: #eef2f8">platform</td>
   <td style="background-color: #eef2f8">object; recommended</td>
   <td style="background-color: #eef2f8">
     A BrandVersion object (see Section 3.2.<em>NEW2</em>) that identifies the user agent's execution platform / OS.</td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">mobile</td>
   <td style="background-color: #dde5f0">integer</td>
   <td style="background-color: #dde5f0">
     1 if the agent prefers a "mobile" version of the content, if available, i.e. optimized for small screens or touch input. \
     0 if the agent prefers the "desktop" or "full" content.</td>
  </tr>
  <tr>
   <td style="background-color: #eef2f8">architecture</td>
   <td style="background-color: #eef2f8">string</td>
   <td style="background-color: #eef2f8">Device's major binary architecture, e.g. "x86" or "arm".</td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">bitness</td>
   <td style="background-color: #dde5f0">string</td>
   <td style="background-color: #dde5f0">Device's bitness, e.g. "64" for 64-bit architecture.</td>
  </tr>
  <tr>
   <td style="background-color: #eef2f8">model</td>
   <td style="background-color: #eef2f8">string</td>
   <td style="background-color: #eef2f8">Device model.</td>
  </tr>
</table>

> BEST PRACTICE: The `browsers` and `platform` fields are optional, but at least one browser or the platform should be provided.  Otherwise, the `UserAgent` object should be omitted.

> BEST PRACTICE: Bidders should not expect any specific order of the `browsers`.  This order is predictable if the `UserAgent` is produced by parsing the *User-Agent* header, but it's random when the source is the CH-UA headers. Bidders that want to detect specific browsers should scan the whole array for entries with well-known browser brand names. We also recommend that all browsers are passed through, without any filtering.

> BEST PRACTICE: The CH-UA specification includes a standard taxonomy for platforms and architectures.  In some cases the identifiers differ from those found in the *User-Agent* header: for example, the ARM architecture is just "arm", not variants like "aarch64" or "armv7l" (the separate `bitness` field replaces most needs of such variants). When a request contains CH-UA headers, exchanges should use the values from these headers, even if the *User-Agent* is also available.  If CH-UA headers are not provided and the `UserAgent` object is constructed via parsing the *User-Agent* header, exchanges are still recommended to adopt the CH-UA taxonomy when applicable. When no standard taxonomy exists in the CH-UA specification for the given individual field of the `UserAgent` object, exchanges should also use the exact strings carried by the UA header; this applies to browser brands and device models.

*It is not a goal of this specification to determine mechanisms for migration from the legacy User-Agent header to CH-UA headers. Exchanges can have different approaches to populate the Structured UA object, e.g. whether to only support CH-UA headers or also support the legacy User-Agent header (parsing it to extract the fields), or which of User-Agent / CH-UA to prioritize if both are available (using the other as a backfill), or how to handle browsers with "frozen User-Agent" where that legacy header has some fields that don't anymore represent the browser precisely, etc.*

### 3.2._NEW2_ Object: BrandVersion

This object identifies a versioned product such as a browser or operating system.

<table>
  <tr>
   <td style="background-color: #893d36">Attribute</td>
   <td style="background-color: #893d36">Type</td>
   <td style="background-color: #893d36">Description</td>
  </tr>
  <tr>
   <td style="background-color: #dde5f0">brand</td>
   <td style="background-color: #dde5f0">string; required</td>
   <td style="background-color: #dde5f0">The common/commercial name of the product.</td>
  </tr>
  <tr>
   <td style="background-color: #eef2f8">version</td>
   <td style="background-color: #eef2f8">array of string</td>
   <td style="background-color: #eef2f8">
     A sequence of version components, in descending hierarchical order (major, minor, …).</td>
  </tr>
</table>
