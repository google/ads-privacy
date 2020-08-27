# Experiment proposal: Structured Geolocation

*The experiment proposal does not reflect plans of any browser, including Chrome, to support similar means of in-browser ad targeting and personalization. While publicly sharing the results of these experiments can lead to productive discussions between ad tech companies as well as browser vendors and privacy advocates on how ad targeting and personalization can be supported with stronger user privacy protections, these discussions may not necessarily lead to browsers building support for the use case or the flow similar to the one proposed in this experiment.*

*The experiment may influence, but does not necessarily represent Google’s current or future real-time bidding product plans.*

## Background
Geolocation is a fundamental signal for RTB; most campaigns include location as a key targeting input. This use case has been historically supported by exchanges by either determining the approximate user’s location (typically using IP geolocation) and sending that information to bidders, or sending the IP address (raw or truncated, depending on an exchange) to bidders, so they can use a third-party IP geolocation service. Exchange-provided location can be more convenient, but the provided taxonomy can also be inconsistent across exchanges, so most bidders tend to prefer to do their own IPGeo lookup. However, passing an IP address for geolocation is challenging from the privacy standpoint, since the IP address is a high-entropy, potentially user identifying signal. To better protect user privacy, exchanges may want to limit user-identifying entropy in bid requests, analogous to the concept of the [Privacy Budget](https://github.com/bslassey/privacy-budget). Fine-grained signals can be "coarsened" in a manner that's incremental and predictable. This experiment proposal offers a more privacy-centric alternative for geolocation signals while allowing for a cross-exchange consistency.

### Existing work

The IAB's OpenRTB protocol specification defines a Geo object that solves some of the problems identified above. It's standardized (no proprietary IDs/databases) and convenient to use, carrying conventional location subfields such as country code, region, city, etc. Some of those fields have international standards, e.g. ISO-3166-1 for country codes, ensuring unambiguous values from all SSPs that communicate the same geolocation. The fine-grained object model, with separate fields for each atom of location data, is much easier to manipulate for privacy or other use cases.

It's not perfect though. Important locations like cities lack standardized codes, and their textual names can have variations of spelling / punctuation (e.g. "Fort Saint James" vs. "Fort St. James"), completeness ("Glarus" vs. "Canton of Glarus"), transliteration ("Gotenba" vs. "Gotemba"), translation inconsistencies ("Kuga District" vs. "Kuga-gun" or "Dagestan Republic" vs. "Republic of Dagestan"), and other problems. Even for standardized fields like region codes and postal codes, different exchanges may sometimes send inconsistent values – or they might be missing any value – due to varied quality or global coverage of location data from different providers.

In addition, the OpenRTB `Geo` object contains fields for the latitude and longitude. Typically, these fields were used for approximate device-reported location.

Because of the potential inconsistencies, bidders often ignore the Geo object completely, preferring to use the IP address field to run their own lookup in a custom or third-party IPgeo database. However, since the sharing of full IP address presents privacy challenges, some exchanges truncate IP address shared in bid requests, which can make bidder-side IP geolocation less granular/accurate: the distribution of IP addresses does not map neatly to geolocation; the same IPv4 /24 block can have IPs spread across distant regions. Exchange-side IP geolocation lookup should be able to use the full IP; the resulting location may be coarsened to protect user privacy, but this coarsening can be predictable (losing precision but not accuracy) because it starts from a consistent IPGeo result.

## Proposal

In addition to political/administrative location fields like region or zip, OpenRTB bid request Geo object includes fields for geographical coordinates: `lat`, `lon`, and `accuracy`. Historically, these fields have been more frequently used for device-reported location. We suggest a method for exchanges to populate these fields and for bidders to consume those even for the cases of non-GPS based location. This approach can accommodate both cross-exchange consistency and user privacy protection goals.

The experiment can be run on a small subset of bid requests, so opted-in biders would expect to see only a minor fraction of their traffic affected. Bidders should maintain the code path that relies on the existing means of geolocation, such as IP geolocation or exchange-specific signals.

### Populating location fields

For experimental requests, if an exchange is able to resolve approximate user location, it can populate the geo field with the usual taxonomic data (`country`, `region`, `city`…), but in addition to those fields, populate approximate `lat/lon` representing that location, including requests where IP address is the primary user geolocation data source.

- Latitude/longitude fields can be populated with a reference latitude/longitude for the most specific approximate location represented by the other fields. For example, if an exchange provides the `zip`, then `lat/lon` will refer to that zipcode; if it only provides location up to the `city`, then `lat/lon` will be the reference position of that city, etc.

- The `accuracy` field can be populated with the approximate confidence radius for the provided location. For example, if the location is New York City (area = 1,583km²), `accuracy` will be 22,447 = radius of a circle with that area in meters. If a specific zipcode of NYC is provided, e.g. 10011 (area = 3.622 km²), then accuracy will be 1,073.

- If the `lat/lon` are provided, bidders should use those approximate coordinates for geotargeting. If the `lat/lon` fields are not available, bidders should continue using the existing code path for geotargeting (i.e. rely on the IP address as provided by exchanges, or the existing exchange-specific geolocation signals).

- If the bidder currently uses IP geolocation or exchange-specific signals, and when `lat/lon` is present, bidders can run *both* geo-targeting code paths, but use only the results of the new code path (based on the approximate `lat/lon`) for targeting. The bidder can then compare location results from either method to produce discrepancy statistics, for example:

	- Count how many requests resolve to the same city vs. different cities.

	- Calculate the linear distance between exchange-provided approximate latitude/longitude and the coordinates provided by the bidder's own geolocation service. The bidder can then provide feedback on the fraction of requests that have this distance significantly above zero, or the average distance across all requests. A bidder may choose to log the relevant fields for requests that are found to have unacceptable discrepancy for further debugging.

### Success criteria

- Bidders receive approximate `lat/lon`, incl. cases location is not derived from device-reported signals; bidders use those for geotargeting of requests, and the average discrepancy stays low – so any effects on campaign targeting should be small or negligible.

- Campaign geotargeting remains accurate (for example, as measured post-impression) on the experimental slice of inventory, and no significant discrepancy is observed between bid request geolocation and geolocation detected at impression time. Auction wins and advertising spend metrics should not show a significant change between experimental traffic (with approximate `lat/lon` field populated) and control traffic (with the existing geolocation fields populated).

### Using approximate coordinates for geotargeting

The approximate latitude/longitude fields, populated as above, can be seen as an *alternate representation for the same location* sent in the fields from `country` to `zip`. The advantage of latitude/longitude is that these are a standard "language" for location; if a bidder doesn't want to deal with taxonomical inconsistencies between exchanges for `city`, `zip`, etc., it can ignore those fields and map the latitude/longitude to a full record from their preferred geolocation / geocode service, or perhaps use that more directly to look up campaigns targeted at the relevant region.
This makes the approximate latitude/longitude a good potential alternative for the IP address for geotargeting purposes.

- Exchanges should send "reference coordinates" for each location. One possible option is the official latitude/longitude made available by public records. In the absence of that, a region centroid computed from maps is another possible choice. There are multiple ways to find a location record from the latitude/longitude fields:

	- The bidder can perform a spatial search filtered by coordinates and location type. The location type can be derived from the finest-grained of the `country`…`zip` fields present in the `Geo` object; for example, if the request contains the `city` but doesn't contain the `zip`, then the provided `lat/lon/accuracy` represent that city. In this case, the bidder can make a query for the location record of type = CITY that has the closest coordinates to the `lat/lon` values (minimum linear distance).

	- Alternatively, the bidder can use a "pure" spatial search, using only the latitude/longitude and the `accuracy`. Campaigns can be spatially indexed by desired geotargeting, so that the bidder could query for campaigns that intersect with the request's geolocation accuracy circle.

	- The bidder might be tempted to make a simple lookup of a geolocation using the `lat/lon` values; this may work if its geolocation database uses the same reference coordinates as the exchange's for the same location. That should only be used as an optimization to short-circuit a full spatial lookup. Even when the exact lookup by reference location works, it can be ambiguous (e.g. a city might have the same official latitude/longitude as its central zip code).

- Exchanges and bidders should agree on the coordinate precision sufficient for approximate geotargeting; coordinates need not be more precise than what would be sufficient to identify a town or a zipcode that is shared by a sufficiently large number of users to protect privacy.

- Provided latitude/longitude should not be interpreted as a precise location of the user's device, irrespective of the number of significant digits; coordinates only indicate the approximate location of the geographical region common to many users, with the provided `accuracy` radius.

- Proposed experiment is specific to IP-based geolocation (`Geo.type=2`), although the same approach could be used consistently if exchanges have access to other sources of location data.

With latitude/longitude fields populated by exchanges using the guidelines above, bidders would have access to a simple, consistent cross-exchange mechanism to obtain geolocation, without the need to rely on IP address for geolocation resolution.

### Additional privacy protections

Geolocation expressed as approximate latitude/longitude coordinates can support additional user privacy protections if needed. For example, if an exchange determines that a given bid request might expose too much user-identifying entropy, then in order to better protect user privacy, an exchange could choose to coarsen the latitude/longitude location by adding noise and increasing `accuracy` radius, as well as to redact some of the `Geo` object fine-grained fields such as `zip`.

