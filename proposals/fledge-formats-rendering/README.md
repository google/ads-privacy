# Requirements

*   **Both the buyer and seller should contribute to the code in the rendering iframe**
    *   Different from HTML ad, both native ad and video ad need contribution from buyer and seller to render an ad. In native ads, the seller (and/or publisher) provides the styles, and the buyer provides the native ad assets (such as images and text). In video, the seller (and/or publisher) provides the ad video player, and the buyer provides the video ad (expressed, perhaps, as a VAST document). A workable solution needs to allow both buyer and seller to contribute to the contents of the rendering iframe.
*   **Assume no changes to the current (Feb 2024) Protected Audience API.**
    *   Given that 3rd party cookie phase out is imminent, we would like to minimize dependency on any API changes that are not yet implemented.
