# Chrome Topics Representation in OpenRTB

## Overview
[Topics](https://privacysandbox.com/intl/en_us/proposals/topics) ([Explainer](https://github.com/patcg-individual-drafts/topics)) is a Chrome proposal that allows ad tech companies to learn coarse-grained advertising topics that a user may be interested in, based on the user’s previous visits to different domains and the topics these domains are associated with. This document proposes a way to represent Topics in OpenRTB protocol.

## Proposal
[Chrome’s Topic API](https://privacysandbox.com/intl/en_us/proposals/topics) may return a list of topics to the caller of document.browsingTopics(). Each topic will have the following attributes.

| Attributes  | Type        | Description |
| ------------ | ----------- | ----------- |
| Taxonomy Version  | String       | Represents the version of the topic’s categories. |
| Model Version   | String        |  Represents the version of the classifier model used to categorize sites. |
| Topic   | Integer        |  Represents the Topics ID from the taxonomy identified by the Taxonomy Version. |

OpenRTB `BidRequest.user.data` can be used to represent Topics.
- `data.name` will have the domain that called the Topics API as the data source
- `data.id` will not be filled.
- `data.ext` will contain an object with the following fields:

| Attributes  | Type        | Description |
| ------------ | ----------- | ----------- |
| `segtax`  | Integer       | A taxonomy enumerated in the [segtax extension](https://github.com/InteractiveAdvertisingBureau/openrtb/blob/master/extensions/community_extensions/segtax.md) |
| `segclass`   | String        |  Classifier version, as provided by the browser |

OpenRTB `segtax` 600 indicates `taxonomy_version` 1. New Topics taxonomy versions will receive new IDs in the ["Vendor specific Taxonomies" list](https://github.com/InteractiveAdvertisingBureau/openrtb/blob/master/extensions/community_extensions/segtax.md#approved-vendor-specific-taxonomies) based on the existing process, optionally from a pre-allocated block.

The `data.segment` array will contain a list of Topics available in the browser to an advertising exchange. Each element in the `segment` array can hold a single topic. The order of topics in the segment array is not specified. The array should not contain duplicate topics.
- `segment.id` will be the topic ID.
- `segment.name` will **not** be set
- `segment.value` will **not** be set


For example, when Chrome returns the following:
- Topic 1:
  - Taxonomy Version: `“tax_v1”`
  - Model Version: `“2206021246”`
  - Topic: `“111”`
- Topic 2:
  - Taxonomy Version: `“tax_v1”`
  - Model Version: `“2206021246”`
  - Topic: `“222”`
- Topic 3:
  - Taxonomy Version: `“tax_v1”`
  - Model Version: `“2206021247”`
  - Topic: `“333”`

Where in the above return, there are two classifier versions and one taxonomy version. The OpenRTB representation can look like the following:

```
{
  ...,
  "user": {
    "data": 
      [{
        "name": "apicaller.com",
        "ext": {
              "segtax": 600,
              "segclass": "2206021246"
               },
        segment: [
            		{ id: "111" },
            		{ id: "222" }
                 ]
       },
       {
        "name": "apicaller.com",
        "ext": {
              "segtax": 600,
              "segclass": "2206021247"
               },
        segment: [
                { id: "333" }
                 ]
       }]
 }
}
```
