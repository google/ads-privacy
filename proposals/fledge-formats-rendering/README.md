# FLEDGE Video and Native Rendering Proposal

This post proposes a rendering approach to render native and video ads using Protected Audience API.

## Scope

This post focuses on how to _render_ native and video ads using Protected Audience API. Other topics such as event tracking, instream video interactions, and format filtering will be covered separately.

# Requirements

*   **Both the buyer and seller should contribute to the code in the rendering iframe**
    *   Different from HTML ad, both native ad and video ad need contribution from buyer and seller to render an ad. In native ads, the seller (and/or publisher) provides the styles, and the buyer provides the native ad assets (such as images and text). In video, the seller (and/or publisher) provides the ad video player, and the buyer provides the video ad (expressed, perhaps, as a VAST document). A workable solution needs to allow both buyer and seller to contribute to the contents of the rendering iframe.
*   **Assume no changes to the current (Feb 2024) Protected Audience API.**
    *   Given that 3rd party cookie phase out is imminent, we would like to minimize dependency on any API changes that are not yet implemented.

# Proposal 1: Seller Specific RenderUrl

The idea is to specify the buyer’s `renderUrl` in the following form:

```
https://seller.com/?burl=https%3A%2F%2Fbuyer.com%2Frender%3Fvideo_ad%3D123
```

*   **Buyer prepends the seller’s format-specific render URL to buyer’s asset URL.**
    *   This allows the rendering frame to first load the seller's rendering code (which could contain an ad video player for video ad or stylings for native ads). 
    *   The seller can get the buyer asset URL from the query parameter and fetch the buyer’s ad assets.


## Example for Video Rendering Flow

_(The following example illustrates the video flow. However, the native flow could be similar.)_


### Set up

Seller hosts:
*   **Seller Rendering Service URL** - The URL pointing to the Seller’s Rendering Service that will return an HTML file which includes an ad video player implementation.

Buyer hosts:
*  **Buyer Asset URL** - A URL pointing to a buyer’s endpoint that will return a video ad’s assets as VAST XML document.


### Join Ad Interest Group

When buyer joins user to interest group, the buyer could create a Full Buyer RenderUrl as follows:



1. Generate a Buyer Asset URL that would return an ad. For example:

```
https://buyer.com/render?video_ad=123
```

2. Prepend the seller’s render URL to the Buyer Asset URL. For example: 

```
https://seller.com/?burl=https%3A%2F%2Fbuyer.com%2Frender%3Fvideo_ad%3D123
```


3. Add the Full Buyer RenderUrl to the interest group.


### Rendering Flow

![Simple prepending - Video Flow](https://github.com/timphsieh-google/ads-privacy/assets/104385842/af9b932c-e4d3-419f-a28d-4bf639a8c618)

1. Seller starts on-device auction with `navigator.runAdAuction()`.  The effective winning render URL may look like the following:

```
https://seller.com/?burl=https%3A%2F%2Fbuyer.com%2Frender%3Fvideo_ad%3D123
```


2. During rendering, the Rendering iframe first sends a request to Seller Rendering Service.
3. Seller’s endpoint at Seller Rendering Service URL returns an ad video player with the information about Buyer Asset URL included.
4. The ad video player can fetch the VAST video ad from Buyer Asset URL and play the ad. 


## Risks to Adoption


### Need one renderUrl per seller

Since the seller is handling the rendering, a buyer would need to create one `renderUrl` for each seller for every ad. As a result, interest groups will need to store O(number of sellers) renderUrls for each ad.. 


### Difficult to support component auction

When a buyer bids into a component auction, either the component seller or the top-level seller need to control the rendering. Each option has its risks. If the top level seller is handling the rendering, then a component seller’s buyer would need to know all of the potential top-sellers that the buyer may bid into. Perhaps, component sellers that the buyer works with could advertise which top-level sellers they submit bids to. The buyer would need to create additional `renderUrl`s for each potential top-level seller. 

If the component seller is handling the rendering, then there needs to be a mechanism for the top level seller to ensure that each component seller properly renders the ad. It may be difficult to verify that a component seller properly applies the publisher’s styling for native ads or use an ad video player that properly report events through Fenced Frame Reporting API. 


# Proposal 2: RenderURL with Seller Rendering Macro

The idea is to specify the buyer’s `renderUrl` in the following form:

```
https://buyer.com/redir?redir_url=${SELLER_VIDEO_RENDERING_URL}https%253A%252F%252Fbuyer.com%252Frender%253Fvideo_ad%253D123
```

*   **Buyer’s <code>renderUrl</code> contains a macro for the seller's render URL.**
    *   The macro <code>${SELLER_VIDEO_RENDERING_URL}</code> allows the seller to insert the URL that points to its rendering endpoint as part of the rendering redirection chain. As a result, the seller can serve native ad styling or video ad player to the rendering iframe. See details in the [later section](Example-for-Video-Rendering-Flow).
*   **Buyer’s <code>renderUrl</code> first points to a redirect service.**
    *   Due to limitations in the Protected Audience API,<code> renderUrl</code>’s macros can be placed only at a <code>renderUrl</code>’s query parameter and not in <code>renderUrl</code>’s URL domain and path. The workaround is to point the buyer’s <code>renderUrl</code> to a simple endpoint that returns a redirect and specify the seller’s rendering URL macro as the query parameter. During rendering, the browser will first request the buyer’s rendering URL. The endpoint will return an HTTP 302 redirect which will then redirect the browser to the seller’s rendering endpoint URL (which, in turn, will render buyer’s VAST video ad or a native ad). 

In addition, the seller may want to verify that the rendering iframe does in fact navigate to its rendering endpoint. Seller can implement an on-device verification script to check that the rendering iframe’s final <code>.contentWindow.location</code> ends up at the seller’s rendering service.


## Example for Video Rendering Flow

_(The following example illustrates the video flow. However, the native flow could be similar.)_

### Set up

Seller hosts:

*   **Seller Rendering Service URL** - The URL pointing to the Seller’s Rendering Service that will return an ad video player.
*   **Optional Seller Verification Script URL** - The URL pointing to a seller endpoint that will return a verification script. The verification script will create a rendering iframe and check that the rendering iframe eventually points to the Seller Rendering Server. If the final location is not correct, the verification script can destroy the rendering iframe. The verification script should have the same origin as the Seller Rendering Service URL.

Buyer hosts:

*   **Buyer Asset URL** - A URL pointing to a buyer’s endpoint that will return a video ad’s assets as VAST.

Any entity (Buyer or any third party) can host:

*   **Redirect URL** - A URL pointing to an endpoint that will redirect the user to a destination URL on the query parameter.


### Join Ad Interest Group

When buyer joins user to interest group, the buyer could create a Full Buyer RenderUrl as follows:



1. Generate a Buyer Asset URL that would return an ad. For example:

```
https://buyer.com/render?video_ad=123
```


2. Generate a Redirect URL that would redirect the user to a destination URL on the query parameter.

```
https://buyer.com/redir?redir_url=
```


3. Use the following template to create Full Buyer RenderUrl.

```
[[REDIRECT_URL]]${SELLER_VIDEO_RENDERING_URL}[[DOUBLE_URL_ENCODED_BUYER_AD_URL]]
```

  * Replace `[[DOUBLE_URL_ENCODED_BUYER_AD_URL]]` with the double [URL encoded](https://en.wikipedia.org/wiki/Percent-encoding) Buyer Asset URL .
  * Replace `[[REDIRECT_URL]]` with Redirect URL. 

For example:

```
https://buyer.com/redir?redir_url=${SELLER_VIDEO_RENDERING_URL}https%253A%252F%252Fbuyer.com%252Frender%253Fvideo_ad%253D123
```

4. Add the Full Buyer RenderUrl to the interest group.

### Rendering Flow

![Add redirect - video flow](https://github.com/timphsieh-google/ads-privacy/assets/104385842/591c2d74-68a4-4154-9027-1b2e13847564)


1. Seller starts on-device auction with `navigator.runAdAuction()`.  The effective winning render URL may look like the following:

```
https://buyer.com/redir?redir_url=${SELLER_VIDEO_RENDERING_URL}https%253A%252F%252Fbuyer.com%252Frender%253Fvideo_ad%253D123
```

2. Seller uses `deprecatedReplaceInURN()` to replace the macro `${SELLER_VIDEO_RENDERING_URL}` with Seller Rendering Service URL for video ad. The effective URL after replacement may be:

```
https://buyer.com/redir?redir_url=https%3A%2F%2Fsellsiderendering.googleads.com%2Fsrs%3Fsformat%3Dvideo%26macro_info%3DSSP_OTHER_DATA%26brurl%3Dhttps%253A%252F%252Fbuyer.com%252Frender%253Fvideo_ad%253D123
```

3. Seller creates a Verification iframe and sets its source to be Seller Verification Script URL. 
4. Seller `postMessage()` the URN returned from `navigator.runAdAuction()` into the Verification iframe.
5. Seller’s verification script creates the rendering iframe and then sets the frame’s source to be the URN from  `navigator.runAdAuction()`.


<img width="469" alt="frames" src="https://github.com/timphsieh-google/ads-privacy/assets/104385842/af5ef7c0-527f-4406-9b83-0c8dbeb9bc73">


6. The Rendering iFrame sends a request to the Redirect URL.
7. Endpoint at Redirect URL should parse the destination URL from the query parameter, decode the destination URL, and redirect the user. The destination URL should be pointing to the Seller Rendering Service.
8. During redirect, the browser effectively sends the following request:

```
https://sellsiderendering.googleads.com/srs?sformat=video&macro_info=SSP_OTHER_DATA&brurl=https%3A%2F%2Fbuyer.com%2Frender%3Fvideo_ad%3D123
```


9. Seller’s endpoint at Seller Rendering Service URL returns an ad video player with information about the Buyer Asset URL.
10. The ad video player can fetch the VAST video ad from Buyer Asset URL and play the ad. 
11. Seller Verification Script can check the rendering iframe’s final location using `.contentWindow.location`. If the final location is not Seller Rendering Server, then the verification script can destroy the rendering iframe. 


## FAQ

### Why do we need a verification script
The seller may want to verify that the rendering iframe does in fact redirect to its rendering endpoint. The on-device verification script is one method to verify this. 

Alternatively, a seller could verify the redirect behavior of the buyer’s ad render URL before allowing a creative to be served. However, if a seller operates as a top level seller, the top level seller may not want to impose the requirement to scan creatives from component sellers. In this case, on-device verification script may be a better option. 


### Why do we need a frame for verification script?

The verification script works by checking whether the last location is the Seller Rendering Service. It checks this by using `rendering_frame.contentWindow.location`. However, this works only if the rendering iframe has the same origin as the verification script’s origin. The publisher’s origin will likely not match the seller’s origin. Creating a new frame to load verification script ensures that the verification script has the same origin as the rendering iframe after the redirect. 


### Can we prepend the macro ${SELLER\_VIDEO\_RENDERING\_URL} to the buyer’s renderURL to avoid a redirect?

As of February 2024, `deprecatedReplaceInURN()` replaces macros only if they are in the query parameters. 


## Risks to Adoption


### Latency

This proposal effectively adds two redirects before rendering the video ad: one for verification script and one for redirect from the redirect service. This would add additional latency to the video rendering. It may be possible to mitigate this by preloading the video ad creatives. 

### Failure to redirect will cause publisher to lose revenue

If the buyer does not redirect back to the seller, then the seller can’t properly count the impression. For example, a video impression is counted after the first frame of the video ad is played. If the buyer does not redirect, then the seller cannot return an ad video player. The seller cannot use the ad video player to trigger an impression. As a result, the seller would not be able to record the impression and charge the buyer. 

### Additional complexity associated with verification

As mentioned in [Why do we need a verification script](#Why-do-we-need-a-verification-script), a seller would like to ensure that a buyer redirects back. However, this also complicates the serving flow. For instance, the seller needs to create another frame for the verification script. The additional complexity increases the chance to have bugs.
