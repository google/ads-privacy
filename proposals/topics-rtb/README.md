# Chrome Topics Representation in OpenRTB

## Overview
[Topics](https://privacysandbox.com/intl/en_us/proposals/topics) ([Explainer](https://github.com/patcg-individual-drafts/topics)) is a Chrome proposal that allows ad tech companies to learn coarse-grained advertising topics that a user may be interested in, based on the user’s previous visits to different domains and the topics these domains are associated with. This document proposes a way to represent Topics in OpenRTB protocol.

## Proposal
[Chrome’s Topic API](https://privacysandbox.com/intl/en_us/proposals/topics) may return a list of topics to the caller of document.browsingTopics(). Each topic will have the following attributes.

| Attributes  | Type        | Description |
| ------------ | ----------- | ----------- |
| Taxonomy Version  | String       | Represents the version of the topic’s categories. |
| Classifier Version   | String        |  Represents the version of the classifier used to categorize sites. |
| Topic   | Integer        |  Represents the Topics ID from the taxonomy identified by the Taxonomy Version. |

OpenRTB `BidRequest.user.data` can be used to represent Topics.
- `data.id` will have a hardcoded string “`chrome_topics_api`” representing Topics API as the data source
- `data.name` will not be filled.

The `data.segment` array will contain a list of Topics available in the browser to an advertising exchange. Each element in the `segment` array can hold a single topic. The order of topics in the segment array is not specified. The array should not contain duplicate topics.
- `segment.id` will be the topic ID.
- `segment.name` will **not** be set
- `segment.value` will **not** be set
- `segment.ext` will contain an object with the following fields:

| Attributes  | Type        | Description |
| ------------ | ----------- | ----------- |
| `taxonomy_version`  | String       | Taxonomy version |
| `classifier_version`   | String        |  Classifier version |


For example, when Chrome returns the following:
- Topic 1:
  - Taxonomy Version: `“tax_v1”`
  - Classifier Version: `“class_v1”`
  - Topic: `“111”`
- Topic 2:
  - Taxonomy Version: `“tax_v1”`
  - Classifier Version: `“class_v1”`
  - Topic: `“222”`
- Topic 3:
  - Taxonomy Version: `“tax_v1”`
  - Classifier Version: `“class_v1”`
  - Topic: `“333”`

The OpenRTB representation can look like the following:

```
{
  ...,
  "user": {
    "data": [
      {
        "id": "chrome_topics_api",
        "segment": [
          {
            "id": "111",
            "ext": {
              "taxonomy_version": "tax_v1",
              "classifier_version": "class_v1"
            }
          },
          {
            "id": "222",
            "ext": {
              "taxonomy_version": "tax_v1",
              "classifier_version": "class_v1"
            }
          },
          {
            "id": "333",
            "ext": {
              "taxonomy_version": "tax_v1",
              "classifier_version": "class_v1"
            }
          }
        ]
      }
    ]
  }
}

```

## Open Questions

- Perhaps the specification should follow the [Segment Taxonomies](https://github.com/InteractiveAdvertisingBureau/openrtb/blob/master/extensions/community_extensions/segtax.md) proposal precisely, as some parties have already begun transmitting this way.  Please note the origin trial period taxonomy has already been assigned segtax 600 and additional segtax id's (it is an enum, not a string) are simple to reserve in blocks from the IAB. In this case, `user.data.id` doesn't typically have meaning, `user.data.name` is the entity reported to have chosen audience membership in the segment. In the case of this segment taxonomy, that would be the entity which called the Topics API. New Topics taxonomy versions would receive new IDs in the ["Vendor specific Taxonomies" list](https://github.com/InteractiveAdvertisingBureau/openrtb/blob/master/extensions/community_extensions/segtax.md#approved-vendor-specific-taxonomies) based on the existing process, optionally from a pre-allocated block.


```
{
  ...,
  "user": {
    "data": [
      {
        "name": "apicaller.com", // the entity which called the Topics API
        "ext": {
          "segtax": 600
          "segclass": "v1" // this is not considered in the Segment Taxonomies / Seller Defined Audiences spec, but seems the appropriate place for it
        }
        "segment": [
          {
            "id": "111"
          }
        ]
      }
    ]
  }
}
```


