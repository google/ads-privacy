# Optimizing Configuration of the Attribution Reporting API
Matt Dawson, Badih Ghazi, Harikesh Nair

## Glossary
This glossary describes terminology that will be used throughout this post.
* Ad-events or ad-interactions: a user action on an advertisement that was shown to them, typically,
* Ad-click: when the user clicks on the ad shown
* Ad-view: when the user views the ad shown (sometimes for a required duration)
* Ad-group: A grouping of ads that share similar targeting settings
* Ad-tech: A technology company which serves online ads and which helps advertisers to plan, buy, measure and/or optimize their ad-campaigns
* [Aggregate Summary Reports](https://github.com/WICG/attribution-reporting-api/blob/main/AGGREGATE.md):  One part of the Attribution Reporting API which provides aggregated summary reports for clicks and views in a privacy-preserving way, without third-party cookies
* Application Programming Interface (or API): A set of software-defined protocols 
* [Attribution Reporting API](https://developers.google.com/privacy-sandbox/relevance/attribution-reporting#what-is-the-attribution-reporting-api) (or ARA): A pair of new APIs that are proposed as part of the Privacy Sandbox to facilitate ad measurement in a privacy-preserving way, without third-party cookies
* Campaign: A set of ad-groups (ads, keywords, and bids) that share a budget, location targeting, and other settings. Campaigns are often used to organize categories of products or services that an advertiser offers
* Click-through conversions: conversions which follow an ad-click over a specific duration
* Conversion attribution: the act of assigning conversion activity to the appropriate prior ad-interaction(s)
* Conversion Type: a description of the conversion, such as a purchase, page view, subscription, etc.
* Conversion Value: The value to the advertiser of a conversion, typically expressed in currency units
* Conversion: a user action which advertisers care about, such as a visit or purchase on a website, which ad-techs report and optimize for on behalf of their advertisers
* [Event-Level Reports](https://github.com/WICG/attribution-reporting-api/blob/main/EVENT.md): One part of ARA which provides event-level reports for clicks and views in a privacy-preserving way, without third-party cookies
* Metadata: additional information related to ad-interactions or conversions
* Privacy-enhancing technologies (or PETs): A set of technologies that facilitate digital advertising in a privacy-preserving way, of which the ARA is an example
* Third-party cookies (or 3PCs): A set of digital identifiers that maintain a persistent identity state for a browser across visited sites
* View-through conversions: conversions which follow an ad-view over a specific duration

In addition to the terms above, some of the descriptions in this post reference technical topics from statistics:
* Aggregation: The process of combining granular pieces of data into larger pieces, typically through the use of an aggregating function such as a “sum”
* [Differential Privacy](https://en.wikipedia.org/wiki/Differential_privacy) (or DP): a framework for extracting useful information from a dataset, while providing protection against leakage of information corresponding to individuals in the dataset; often this is achieved by perturbing the information being extracted. When the perturbation is applied to statistics involving the entire dataset, this is also referred to as Central-DP.
* Expected value: A concept from statistics which describes the “average” outcome of a random quantity
* [Laplace probability distribution](https://en.wikipedia.org/wiki/Laplace_distribution): A description of a particular type of random outcome in which the most likely outcomes lie near the expected value and the likelihood decreases exponentially away from the expected value
* [Local Differential Privacy](https://en.wikipedia.org/wiki/Local_differential_privacy) (or Local-DP): A special case of “Differential Privacy” above, where noise is added to each individual data point to protect data privacy
* [Privacy Budget](https://en.wikipedia.org/wiki/Differential_privacy) or  $\epsilon$ (see “sequential composition”): [not to be confused with the unrelated [Privacy Sandbox Technology](https://developers.google.com/privacy-sandbox/protections/privacy-budget) of the same name] A parameter which controls the degree of privacy a differentially private mechanism can guarantee. For the Laplace mechanism (which is used for Aggregate Summary Reports in the ARA), if multiple queries are used on the same dataset, this privacy budget must be divided across each of them.
* [Randomized response](https://www.jstor.org/stable/2283137): A statistical mechanism by which observations are randomly perturbed in order to satisfy a privacy requirement
* Sensitivity budget: related to “Privacy Budget” above - a fixed integer designated to an ad interaction in the ARA, the allocation of which across aggregations controls relative noise levels. There is a duality between sensitivity budget and privacy budget, in the sense that partitioning sensitivity budget is equivalent to partitioning privacy budget.
* Slice or Node: A particular instance of aggregation such as at the ad-campaign or ad-campaign-country level; also referred to as a “node” when describing slices as elements which comprise the aggregate tree.
* Truncation: The act of transforming a random variable so that its realizations below (analogously above) a predefined threshold are retained, and those above (analogously below) are dropped. When applied to conversions, truncation implies information loss because conversions following an ad-interaction that are beyond a predefined threshold are not observed
* Variance: A concept from statistics describing the deviation of a random quantity from its expected value

## Summary
This post discusses Google Ads’s approach towards improving the utility of the Attribution Reporting API from Chrome’s Privacy Sandbox which we plan to utilize for ad measurement, including on Google-owned inventory such as Search and YouTube, as well  on third-party inventory available via our advertising technology products. It is the second in a two-part series. [Our first post](https://github.com/google/ads-privacy/blob/master/Combining%20the%20Event%20and%20Aggregate%20Summary%20Reports%20from%20the%20Privacy%20Sandbox%20Attribution%20Reporting%20API.pdf) explained our approach to blending Aggregate Summary and Event-level reports to form a cohesive view of conversion activity measured by the ARA API. There, we documented that leveraging both types of reports leads to improved data quality by taking advantage of the strengths of each.

In this post, we highlight another critical aspect of obtaining high quality data from the ARA which is linked to its configuration by the ad-tech. While post-processing and blending (the subject of the first post) concern themselves with cleaning of the data once it has been received by the ad-tech, configuration refers to the steps ad-techs must take prior to receiving any reports. Essentially configuration is the answer to “what data should I query, and how should I query it?” Google Ads has explored this question in depth for its own usage, and our explorations with the ARA have shown that intelligent configuration is important to obtaining optimally accurate attribution data. Further, when implemented thoughtfully, we have found that intelligent configuration can lead to utility gains.

Questions around configuration arise in many parts of the ARA’s usage by ad-techs, which is a testament to its underlying design flexibility. This post does not aim to enumerate and address all possible configurations possible with the API. Instead, we will highlight two particular configurable settings in Aggregate Summary Reports – many-per-click (MPC) limit and sensitivity budget allocation – and share our approach to fine-tuning these settings. Our hope is by explaining our general approach to the problem, and by outlining how it applies to these two key configuration questions, we help ad-techs and ecosystem partners make progress on the configuration question to improve the utility from their own usage of the ARA API. Also, we hope that by sharing our ideas and experiences, more innovation will be spurred in the ecosystem as to how to leverage the flexibility provided by the ARA and configure it appropriately to deliver on the diverse needs of the advertising ecosystem. 

## Short Recap of the ARA
**Note**: This post assumes some knowledge of the nature of the attribution reports generated by the ARA. We discussed this topic in some detail in our [previous post](https://github.com/google/ads-privacy/blob/master/Combining%20the%20Event%20and%20Aggregate%20Summary%20Reports%20from%20the%20Privacy%20Sandbox%20Attribution%20Reporting%20API.pdf). Readers new to the ARA API are advised to read our first post to familiarize themselves with key details of the API. If the reader has sufficient knowledge in this regard, we recommend skipping ahead to the section below titled “The Role of Configuration”.

## The Role of Configuration
The picture below provides a schematic of how Google Ads leverages the ARA data. We configure the event-level and the aggregate summary reports API to obtain “good” raw data from the API; then post-process the two reports to develop a blended dataset that we leverage for ads use cases. Our previous post focused on the post-processing aspects of the Event-level and Aggregate Summary Reports shown in orange in the figure. We made the case that leveraging both report types is important for maximizing the utility potential of the ARA data, and provided a general strategy for how ad-techs can think about leveraging the ARA. 
<p align='center'>
<img width="600" src="https://github.com/google/ads-privacy/assets/139409595/be3e2ea0-0a9f-47b3-90b9-309bc782007b"></p>
This post focuses on the left-hand side of this figure which deals with configuration. When we say that an element of the ARA is “configurable”, we mean that it is a setting under the control of the ad-tech. When we say that an element is “customizable”, we mean that the setting can be set by the ad-tech in specific ways for certain types of its traffic, such as configuration on a per-advertiser basis. 

Why is there a need for configuration and/or customization? In short, it is because the ARA allows for very flexible usage. The flexible nature of the ARA supports a wide variety of measurement use-cases across a rich diversity of ad-techs and advertisers. We view this flexibility as extremely useful and have found that thoughtful consideration of how to leverage this flexibility is a crucial step in ensuring that the data we receive is of the highest quality possible.

As a concrete example, in our previous post, we noted that the maximum allowable number of conversions per ad interaction can be configured with Aggregate Summary Reports. This fact raises some immediate questions: what are the data quality implications of this particular setting? How should we think about optimizing the maximum allowable number of conversions per ad interaction? What would be a general approach towards such optimization? How does this interact with other settings? These are some of the questions we will address in the coming sections.

### Configuration with Aggregate vs. Event-Level Reports
While there are configurable settings for Event-level Reports, this post will focus on Aggregate Summary Reports. There are two main reasons for this, described below.

First, the quality considerations in the Aggregate Summary Reports rely more heavily on configuration than they do on post-processing, and there is much to configure with these reports. In Google Ads’ setup, the event debiasing procedure described in our [previous post](https://github.com/google/ads-privacy/blob/master/Combining%20the%20Event%20and%20Aggregate%20Summary%20Reports%20from%20the%20Privacy%20Sandbox%20Attribution%20Reporting%20API.pdf) contributes most to improving the data quality of Event-level Reports. On the other hand, much of the utility gain we have observed on the aggregate side comes from configuration adjustments. 

Second, we expect that the configuration questions and solutions we raise in this post will be equally relevant to all ad-techs using the ARA. In contrast, configuration of Event-level Reports (such as optimal usage of the conversion metadata bits) tends to be more ad-tech-specific, and is therefore less generalizable. For instance, some ad-techs that primarily serve advertisers who are focused on awareness and consideration campaign goals may want to prioritize usage of the metadata bits to track upper-funnel conversions such as website visits. And ad-techs that primarily serve advertisers who are focused on conversion campaign goals may want to prioritize usage of the metadata bits to track lower-funnel conversions such as add-to-carts and purchases.

### Forming Intuition About Configurable Aggregates
In our previous post, we explained that Google Ads preferred a hierarchical aggregate structure for the Aggregate Summary Reports from the ARA (please see our previous post for rationale and additional background).  By “hierarchical”, we mean a hierarchical “tree” of aggregates where nodes higher up on the tree contain coarse aggregates; with those nodes having children that are slightly more granular, and ultimately with the leaves of the tree containing the most granular aggregates. The tree analogy is a useful tool for building intuition as we dive deeper into the question of how to obtain high-quality aggregate data.

Recall that the ARA introduces noise and truncation to the underlying conversion data in order to protect user privacy. This means that the data we receive will not be 100% accurate. More importantly, granular nodes and leaves tend to be more impacted than coarse aggregate nodes near the top of the tree. In the presence of noise and truncation it may not be possible to achieve highly accurate aggregates at highly granular slices for all advertisers. This is a key tradeoff to appreciate.

For example, consider the following tree which is an example partner from our application in Part 3. Each node on the tree represents a particular aggregate slice, with the true conversion count shown in bold. Darker nodes correspond to those which are less susceptible to the addition of noise in Aggregate Summary Reports.
![Sample Key Hierarchy](https://github.com/google/ads-privacy/assets/139409595/663babd9-07b6-402c-b370-3704b34c01b1)

There are three key ideas to keep in mind when obtaining data in such an environment:
1. First, aggregate data under truncation and noise implies a **granularity vs. accuracy tradeoff**. Granular slices are typically more valuable than coarse ones as they provide more detailed conversion measurement (i.e., higher utility), but they are less likely to be measured with high accuracy due to the noising induced by the API (i.e., lower accuracy).
2. Obtaining good data is **about striking a graceful balance** between these two forces. This balance can be struck by **effectively utilizing the configuration levers which are provided by the ARA**; we will discuss these levers and their importance in detail.
3. Each advertiser has different data characteristics and what works best will vary from advertiser to advertiser. This is where **customization plays an important role**.

As a simple motivating example to highlight these three points, consider two advertisers, A and B. Suppose that advertiser A has many thousands of conversions and advertiser B has very few. The mechanics of the ARA suggest that “thin” slices (those with low conversion counts) will have a lower signal to noise ratio and thus be more susceptible to noise. On the other hand, “thick” slices (those with high conversion counts) will have a higher signal to noise ratio and thus be less susceptible to noise, relatively speaking. In this case, advertiser A may be able to collect granular data with reasonable accuracy because they have many conversions. On the other hand, the same granular slices for advertiser B may be compromised by noise since the slices will be much thinner. This basic scenario sets the need for intelligent customization of at least a few configurable settings. For instance, advertiser B may be better off collecting more coarse aggregates so that their data is more accurate, at the cost of granularity. 

In what follows, we will describe some specific mechanisms where optimizing configurations this way is useful. We will also share Google Ads’ approach to resolving the questions we posed above of configuration and customization. Finally, we will provide a concrete example to showcase the efficacy of our approaches. While the exact implementation may vary for each ad-tech, we believe the fundamental tradeoffs mentioned above will need to be addressed by all.

## Part 1: What is Configurable in Aggregate Reports? Why Does Configuration Matter?
We now discuss what is configurable in the Aggregate Summary Reports. Specifically, we introduce two configurable settings - the MPC limit (also referred to as “[repeat rate](https://support.google.com/google-ads/answer/3438531?hl=en#:~:text=Your%20repeat%20rate%20is%20the,the%20%E2%80%9Cone%20conversion%E2%80%9D%20setting)" limit) and the Sensitivity Budget Allocation - which from our perspective, represent the first-order configuration consideration for ad-techs.  Part 1 describes these configuration tasks along with key details of how conversions are registered with the Aggregate Summary Reports. Part 2 outlines our approach to making configuration decisions in these two areas. 

### Two Major Configuration Questions: MPC Limit and Sensitivity Budget Allocation

#### How Noised Aggregation is Achieved in the ARA
Before diving into the configuration details, it is important to understand exactly how aggregation works for Aggregate Summary Reports. For a comprehensive description, please see this [documentation](https://github.com/WICG/attribution-reporting-api/blob/main/AGGREGATE.md#attribution-trigger-registration). This is a four-step process described below:
![Scaling Process](https://github.com/google/ads-privacy/assets/139409595/992570b9-2632-420e-bd7d-c60179047564)

Briefly, the process used by the ARA to noise the conversion data involves first scaling up the conversions by a scaling factor; adding up the scaled conversions into an aggregate; adding noise to the aggregate; and then rescaling the noised aggregate to the original units. The first panel of the picture above describes these four steps and the blue lower panel provides a concrete example of how this works.

Let’s build some intuition about the implications of this process. First, note that if there were no noise (i.e., if the noise, L, were to equal 0), this process would simply yield the original conversion count: we would scale the conversions up, aggregate, then scale them back down. Next, note that noise is applied in the scaled space and that it is always the same type of noise. By this, we mean that while the added noise is a random quantity (and therefore cannot be predicted perfectly), it is always the same type of random quantity in terms of its distribution (i.e., from a Laplace distribution with fixed parameters). Finally, note that the effective amount of noise is determined by the scaling factor: the larger the scaling factor, the smaller the effective amount of noise.

The last point is an important one. It implies that the effective amount of noise in Aggregate Summary Reports can be controlled to some extent by the ad-tech, by simply modifying the scaling factor, which is under the control of the ad-tech. To see this, note that in the example above the noise component of our observed aggregate is given by L / 100, which, given the Laplace distribution, is a random quantity with standard deviation given by $\frac{1}{100}\cdot\frac{\sqrt{2}L1}{\epsilon}$. If we were to increase the scaling factor to 1000, the resulting standard deviation would be $\frac{1}{1000}\cdot\frac{\sqrt{2}L1}{\epsilon}$, corresponding to a 10X decrease in noise. Therefore, one key aspect of obtaining accurate aggregate data is to maximize the scaling factor.

One may then naively conclude that by doing this, an ad-tech can drive the noise practically to zero. However, the ARA includes a guardrail to prevent this. There is a built-in constraint on the size of the scaling factor. Each ad interaction is given a sensitivity budget of L1 (current value of L1 is set to $2^{16}$ in Chrome). This means that the sum total of all scaled contributions for any ad interaction is constrained to be no more than $2^{16}$. (Note also that the noise added is proportional to this number.) For what comes below, note that the sensitivity budget of $2^{16}$ is for each ad interaction.

Now that we understand more clearly about how aggregation and noising works with the ARA, we are ready to discuss how ad-techs can work within these constraints in order to maximize the quality of their data.

#### Configurable Setting: MPC Limit
It is common for advertisers to have interest in multiple conversion actions following an ad interaction. For instance, if an ad click leads to multiple purchases over a given time period, the advertiser (and in turn, the ad-tech on the advertisers’ behalf) may be interested in capturing all of these conversions, rather than just one of them. This information may be used to report on conversions that followed the ad click or to train models of conversions following ad clicks that are useful for bidding and budgeting automation. We refer to this as a many-per-click (MPC) setting. A key thing to note is the aggregation procedure described above has important implications for capturing this kind of conversion behavior.

Recall, we emphasized above the sensitivity budget L1 is defined at the ad interaction level, and importantly, not the conversion level. This means all conversions following a single ad interaction must share the L1 budget. In addition, since per-conversion contributions are due at the time of conversion registration, the ad-tech must pre-specify them prior to the conversion occurring (i.e., at the time of ad-interaction). This means that the ad-tech must explicitly make a decision as to the maximum number of conversions they wish to register against a given ad interaction.

To understand the implications of MPC choice, let's look at the figure below.

![MPC example](https://github.com/google/ads-privacy/assets/139409595/22505bb2-ce73-41cb-9e53-b143cc2676c4)

The figure above depicts two identical click-conversion scenarios under two different MPC settings. Under the first setting (top panel: MPC = 5), two conversions are dropped for the second click because the sensitivity budget has been exhausted, leading to truncation. The second setting (bottom panel: MPC = 7) is able to capture all conversions, but at the cost of more noise since each conversion is forced to use less of the sensitivity budget.

Immediately we can see why registering multiple conversions is at odds with the goal of maximizing contributions: the more conversions we wish to register, the less per-conversion budget we have for contributions. In other words, we see that **MPC choice involves an explicit tradeoff between truncation (i.e., conversion coverage) and noise (i.e., accuracy), where if we want to reduce one of these, we are forced to increase the other**. For example, changing the MPC limit from 1 to 10 will allow for more conversions to be registered, but must come at the cost of 10x more noise (since contributions per conversion must be reduced by 10x). 

We have arrived at our first open question about configuration: if we are interested in registering multiple conversions per ad interaction, what should we select as our MPC limit? What MPC limit strikes a graceful balance between coverage and accuracy?

#### Configurable Setting: Sensitivity Budget Allocation
Next, we discuss  another key situation in the ARA where contribution “splitting” arises. This issue relates to our chosen hierarchical structure. Specifically, in a hierarchical structure, each conversion can contribute to each level of the hierarchy and contributions must therefore be split across each of the hierarchy levels. Let us discuss this in more detail below.

Suppose that our desired aggregate structure has N levels, where each level is created by appending some additional dimensions to its parent level. A simple example of this is depicted below, where we can define:

* Level 1: Campaign, Conversion Type
* Level 2: Campaign, Conversion Type, Ad Group

<p align='center'>
<img width="600" src="https://github.com/google/ads-privacy/assets/139409595/01dc7992-6c7f-4b94-b82f-1f97aabd6bc5"></p>

In this example, the key dimension “Ad Group” makes Level 2 more granular than Level 1. As we have mentioned previously, the hierarchical structure is Google Ads’ preference, which we consider to be a useful construct. But it’s not a requirement, and in fact the ARA does not make any distinction between hierarchical and non-hierarchical structures when it aggregates: it simply counts contributions for particular aggregate buckets.

That is, the ARA views the above hierarchy in the following way when constructing Aggregate Summary Reports:
* Bucket 1: Campaign == 1, Conversion Type == Sale
* Bucket 2: Campaign == 2, Conversion Type == Sale
* Bucket 3: Campaign == 1, Conversion Type == Sale, Ad Group == 1
* Bucket 4: Campaign == 1, Conversion Type == Sale, Ad Group == 2
* Bucket 5: Campaign == 2, Conversion Type == Sale, Ad Group == 3

When a conversion happens, the allocated contributions will be placed in the bucket(s) with matching attributes, later to be aggregated with other contributions and noised as described above. Note, however, that it is possible (and expected, under a hierarchical setup) for conversions to match multiple buckets. For example, if a conversion on Campaign 1, of Conversion Type Sale, and on Ad Group 1 occurs, it will match both Buckets 1 and 3. If we want the conversions to contribute to all matching buckets, we must split the conversion’s contributions across those buckets. Practically, this means we are splitting a conversion’s contribution across all levels of the aggregate hierarchy.

The allocation of contributions across hierarchy levels need not be uniform. This fact brings us to our second important configuration decision: how should we allocate conversions’ contributions across these levels? While it is true that uniform allocation will yield uniform noise levels, that may not be desired. To see this, recall that aggregate levels near the top of the tree are coarser than nodes and leaves further down the hierarchy. This means they will also have more conversions and therefore may be able to tolerate higher noise levels, suggesting that allocations with increased contributions on lower levels may be preferred. 

One way to reason about this is by envisaging the concept of a useful tree for an ad-tech; i.e., a tree of aggregates for which the signal to noise ratio of its nodes is within acceptable/useful limits. By allocating more sensitivity budgets to some nodes of the original aggregate tree, we make those nodes less noisy, and in turn, we change the shape of the useful tree. With this lens, we can therefore view the sensitivity budget as a tool in the hands  of the ad-tech to determine the width and depth of its useful aggregate tree. However this allocation is not infinitely free for the ad-tech, because there is a limit on the total sensitivity budget that can be allocated. Changing the shape of the useful tree by allocating budget on some nodes takes away the budget from others, leading to hard tradeoffs on usefulness across slices that need to be resolved. 

To summarize the discussion of contribution allocation and MPC limits, we have that the conversion contributions for a particular aggregate level are equal to $\frac{L1}{\text{MPC limit}}\cdot \text{fraction allocated for level}$, where "MPC limit" and "fraction allocated for level" are configurable settings. Under this framing, it is clear that the MPC limit and sensitivity allocation are intimately connected.

### Customization
We have motivated why deliberately approaching the configuration questions above is important for improving the quality of data in the Aggregate Summary Reports. A next critical layer is how these configuration decisions are decided on a more granular level, such as on a per-advertiser basis.

The motivation here is that while each advertiser may measure similar quantities, the characteristics of their conversion data may differ dramatically. For example, if we consider the question of optimal MPC limits, we expect to find different answers for retail vs. travel vs. subscription-based businesses, each of which may have a different profile of the number of conversions that follow an ad exposure, necessitating different optimal MPC choices for each. We call this “customization.” Motivated by this, we encourage ad-techs to evaluate the needs of their advertisers given the constraints imposed by the ARA and to consider a flexible, customized approach to ensure high quality data collection on behalf of all advertisers.

## Part 2: How Do We Achieve Optimal Configuration?
Now that we have defined the configuration (and customization) questions, we can begin to discuss our approach for choosing good configurations. Our general approach is simple and is comprised of two steps:

1. We first define an objective function which allows us to quantify the tradeoffs mentioned in Part 1. By “objective function”, we mean a mathematical characterization of quality which we wish to maximize. 
2. We choose the specific configuration values which optimize this function.

Essentially what we are doing with this approach is converting the open questions posed above into precise optimization problems. With this framing, we are able to systematically discover the configurations which are most likely to yield high quality Aggregate Summary Reports. This section describes details of our approach at a high-level.

### Defining Our Objective (RMSRE)
A crucial step in any optimization problem is to first define the quantity of interest - what is it that we are trying to optimize? For example, ad campaigns may try to optimize for conversion volume or return on investment; commuters may try to optimize for distance traveled or commute time, and so on. We refer to these quantities as objective functions and it is imperative that we define an objective function for Aggregate Summary Reports on the basis of which we can make quantitative assessments about which configurations are best.

Our reasoning is as follows. In the case of Aggregate Summary Reports, the objective function should be related to accuracy - that is, we should desire that the data we see in the reports is as close as possible to the underlying ground truth. To that end, we start with some basic definitions (we will expand on these later):
* $x$: the ground truth conversion count for a particular aggregate slice
* $Y$: the observed conversion count from Aggregate Summary Reports for that same slice
With these definitions, it is clear that we want the difference between $x$ and $Y$ to be minimized, but there are several ways we can characterize their difference. For example, we can take $|Y-x|$, or $(x-Y)^2$, or $\frac{Y}{x}$, etc. 

We have explored several options of quantifying this difference and have arrived at a function called Root Mean Square Relative Error (RMSRE). We have introduced this function in our related technical paper on optimizing hierarchical queries; this metric is also used in the Noise Lab to characterize differences between ground truth and ARA simulated data. So, what is this function? Define,
$$RMSRE = \sqrt{E\left[\left(\frac{Y-x}{x}\right)^2\right]},$$
where $E$ stands for expected value and is taken over the randomness in $Y$. Let us break down this metric, starting from within the parentheses:
* $\frac{Y-x}{x}$ is the difference between ground truth and reported conversions, relative to the ground truth. By ground truth here, we mean the true number of conversions in that slice. We consider this as a useful starting point because aggregate slices can vary dramatically in size. For example, a difference of 5 conversions is likely not a big problem if the ground truth is 1,000 but is quite a big difference if the ground truth is 2.
* Squaring the relative error allows for a bias/variance decomposition which we will highlight in the next section.
* The expectation is needed since the API applies random noise to the data. This means that the relative error is not deterministic, but the expectation is there to compute a “typical” value of the squared relative error.
* The square root is there to counteract the square so that this metric can (loosely) be interpreted as the expected relative error between ground truth and observed (and not the square of this).

One known limitation of relative errors is their susceptibility to explode when x is near 0 (in this case, when we are examining aggregate slices with very low conversion counts). A small modification we have made to prevent this explosion is to include protection in the denominator:
$$RMSRE_T = \sqrt{E\left[\left(\frac{Y-x}{\max(T, x)}\right)^2\right]},$$
where $T$ is chosen to be some small number; in our applications, we will use $T=5$, and it could vary by ad-tech and context. Finally, while this objective function can be applied to a single aggregate slice, we can understand the full picture of an advertiser’s aggregate tree by looking at averages (or weighted averages) of $RMSRE_T$ across all slices.

The last point we will make in this section is a crucial one. It is that $RMSRE_T$ as defined above depends on the configurations described in Part 1. That is because those configurations control what we observe in $Y$ - both in terms of noise (more specifically, the effective variance of the noise added to aggregates) and truncation. This means that we may write $RMSRE_T(configs)$ and choose the configs which minimize this function. This summarizes our general optimization approach.

### Optimal MPC Limits
Our first objective is to ensure that we are attempting to register the correct amount of conversions per ad interaction. For this, we will use $RMSRE_T$ as the guiding objective, where since $RMSRE_T$ depends on the selected MPC limit $m$, we may write $RMSRE_T(m)$ and minimize this function with respect to $m$, for a fixed budget allocation.

One of the nice properties of this function is that it has a particular bias/variance decomposition, which is precisely what we aim to balance with MPC optimization. To see this, first define $x(m)$ to be the MPC-truncated conversion count for the slice for a given choice of $m$. Then we have,
$$E\left[\left(\frac{Y-x}{\max(T,x)}\right)^2\right] = \frac{1}{\max(T,x)^2 }E[(Y-x)^2] = \frac{1}{\max(T,x)^2 }[(x(m)-x)^2 + Var(Y)].$$

What does this mean? Simply put, our objective function can be broken into two components:
1. The impact of truncation: $(x(m)-x)^2$
2. The impact of noise: $Var(Y)$, since the variance of the noise depends on $m$.
Thus, $RMSRE_T$ naturally tries to limit the impact of truncation, while simultaneously avoiding a variance blow up by making conversion contributions too small. Note also that since the noise distribution is known, $Var(Y)$ can be understood analytically (that is, there is a formula based on the [Laplace distribution](https://en.wikipedia.org/wiki/Laplace_distribution)).

Let’s return to our three-click example from before to see how this optimization works. 

<p align='center'>
<img width="400" alt="bias" src="https://github.com/google/ads-privacy/assets/139409595/dbe3d182-5642-4a79-ba1a-ed6ac820abf1"></p>

Suppose that we wish to aggregate the conversions belonging to these clicks (for example, if they correspond to the same aggregation slice). In this case, we have: $x=14$ but we saw under $m=5$, we will have $x(m)=12$. That is, there are 14 conversions total, but if we only allow up to 5 conversions per click, we would observe 12 conversions after aggregating. We have already mentioned that increasing $m$ to 7 will ensure that there is no truncation, but it will come at the cost of noise. 

Let’s see how this plays out when we aggregate this slice (using $\epsilon=1$ as an example):

<p align='center'>
<img width="400" alt="bias" src="https://github.com/google/ads-privacy/assets/139409595/9fabbd1a-8b39-46a6-b647-029a9aa661a5">
<img width="400" alt="variance" src="https://github.com/google/ads-privacy/assets/139409595/fc67b446-7ffa-49e5-a508-03a8ce22384f">
<img width="400" alt="rmsre" src="https://github.com/google/ads-privacy/assets/139409595/62e6a2ca-d65e-4f62-9b8f-6eaa728fa48b"></p>

In the plots above, notice that the bias component goes to 0 as long as we set $m$ to 7 or larger, as for these choices of MPC limit, we would be able to avoid any truncation. However, the variance component grows with $m$. For example, when $m=7$, the noise variance is 98 (corresponding to a standard deviation of 9.9 conversions) which is large relative to the slice’s total conversion count of 14. Finally, $RMSRE_T$ finds a graceful balance between these two components, and recommends an MPC limit of 4, which is essentially trying to minimize the noise level by allowing for some truncation. Viewed this way, we can see that the MPC problem is a specific example of the classical bias-variance tradeoff from statistics. This example is using $\epsilon=1$, and one would find different tradeoffs under different values of $\epsilon$.

### Optimal Sensitivity Budget Allocation
The next optimization we consider is how to allocate budget across the levels of an aggregate hierarchy. Recall the example we provided of such a hierarchy which we use in the Part 3:
![Sample Key Hierarchy](https://github.com/google/ads-privacy/assets/139409595/663babd9-07b6-402c-b370-3704b34c01b1)

In Part 1, we highlighted the fact that under a hierarchical setup, conversions will fall into several aggregation slices. Given this fact, we must decide how we wish to allocate the sensitivity budget across those slices. Again we face a tradeoff: leaf nodes (at bottom) contain the most information since they are the most granular, but their granularity means that they are also more sensitive to noise.

To solve this problem, one can also use $RMSRE_T$ as the objective function, but in this case we consider the objective as a function of $b$ which is a **vector** of fractions which correspond to sensitivity budget per level. For example, in the four-level hierarchy depicted above, using $b=[0.25, 0.25,0.25,0.25]$ corresponds to splitting the sensitivity budget equally across all four levels of the hierarchy, while $b=[0.01, 0.97, 0.01, 0.01]$ corresponds to putting most of the budget at the second level. In this way, we again can write $RMSRE_T(b)$ and find the value of $b$ which provides the most accurate tree as a whole (for example, taking the average $RMSRE_T$ over all nodes or something similar). The optimal solution can be found by standard constrained optimization algorithms or in small cardinality situations, by brute-force enumeration.

Generally, allocating more budget toward the bottom of the tree is required to ensure that the thinnest slices can be measured accurately. If we use the post-processing technique from our [first post](https://github.com/google/ads-privacy/blob/master/Combining%20the%20Event%20and%20Aggregate%20Summary%20Reports%20from%20the%20Privacy%20Sandbox%20Attribution%20Reporting%20API.pdf), information can flow up the tree as well, so that children nodes can improve their parents; this is a force towards allocating sensitivity budget towards lower nodes. Also, it shows the interaction between post-processing techniques and configuration - using them both in tandem can yield substantial benefits. 

We have also found that sometimes there are circumstances where very granular measurement is not possible, such as cases where the tree is very sparse, the optimal MPC is very large, or when  is small. In these cases, it may be useful to set a minimum quality bar, where if a slice cannot reach this bar under any budget allocation, the slice is abandoned and the budget is allocated at a less granular level. 

### Optimization Post-3PCD
The optimization approach we described above assumes knowledge of ground truth conversion counts which may not be realistic. However, we have found that looking at historical API data can serve as a useful proxy for conversion volumes, even when the historical data is coming from naively configured ARA reports. In addition, Chrome has proposed a [mechanism](https://github.com/WICG/attribution-reporting-api/issues/705) for debug information post-3pcd which may be useful for configuration (particularly for MPC configuration). However, this proposal has not been finalized at the time of writing of this post.

## Part 3: An Example
We now turn to an example to demonstrate the effectiveness of optimized configuration on a public dataset provided by Criteo (see [Criteo Sponsored Search Conversion Log Dataset](https://ailab.criteo.com/criteo-sponsored-search-conversion-log-dataset/) for details).  Our results show that partner-specific customized configurations are beneficial in terms of data accuracy.

### Data Description
The dataset is click-level, meaning every row is a click with some click and conversion features for 287 Partner IDs (partner corresponding to the product seller). While the original conversion column “Sale” is binary, we have augmented this dataset to include the possibility of multiple conversions per click. To achieve this, for every sale, we sampled from a zero-truncated Poisson distribution with rate dependent on the Partner ID’s overall conversion rate. We also omitted clicks with 0 sales for the sake of simplicity. Below is a small sample of the event-level data for this demonstration.
  
| partner_id | product_country | device_type | product_age_group | number_of_conversions |
| ---------- | --------------- | ----------- | ----------------- | --------------------- |
| 46         | 5               | 2           | 5                 | 1                     |
| 1          | 1               | 2           | 1                 | 5                     |
| ...        | ...             | ...         | ...               | ...                   |
| 80         | 2               | 3           | 1                 | 8                     |

From this data, we obtain a four-level aggregate tree for every partner as depicted previously in this post, and the goal of this experiment is to quantify the difference between data quality under naive vs. optimized configuration.

### Baseline Configuration
We begin by setting some informed but relatively naive settings for both the MPC limit and sensitivity budget allocation. For MPC, we set a global MPC limit of 15 for all advertisers, based on the fact that most conversions will be retained with such a limit, according to the graph below:

<p align='center'>
<img width="500" alt="baseline_dist" src="https://github.com/google/ads-privacy/assets/139409595/3e111e43-c7e2-46ec-b637-ed82d2fa90ab"></p>

For sensitivity budget split, we take $b=[0.25,0.25,0.25,0.25]$ as the baseline, splitting sensitivity budgets equally across the four aggregation levels.

### Customized Configurations
When we examine this dataset on a per-partner basis, we find that there are substantial differences between advertisers. For example, consider the comparison below in terms of conversions per click:

<p align='center'>
<img width="500" alt="comparison" src="https://github.com/google/ads-privacy/assets/139409595/bfd79575-73b4-48e6-9d60-1bd1f618a2de"></p>

Partners 241 and 85 have dramatically different conversion behavior, where the baseline MPC limit of 15 is larger than necessary for Partner 241, while for Partner 85 the same limit would result in non-trivial conversion truncation. We customize the MPC on a per-partner basis by choosing the minimum m such that there is no resulting truncation. 

In terms of sensitivity budget, we consider the following allocations:
* $[0.25,0.25,0.25,0.25]$
* $[0.97,0.01,0.01,0.01]$
* $[0.01,0.97,0.01,0.01]$
* $[0.01,0.01,0.97,0.01]$
* $[0.01,0.01,0.01,0.97]$
  
We use each allocation and apply the one which yields the most aggregate slices of high quality, where we set a threshold of $RMSRE_T < 0.1$. Again, this optimization is done separately for each advertiser.

### Results
Our results show that optimizing these configurations leads to large improvements in utility for all values of $\epsilon$. For example, for $\epsilon=20$, the average fraction of high quality slices jumps from ~84% under the naive config to ~98% for the optimized configuration.

<p align='center'>
<img width="500" alt="results" src="https://github.com/google/ads-privacy/assets/139409595/8163feb4-d695-4e2c-b612-9a539c42d80d"></p>

Naturally, the exact magnitude of this difference will depend on several ad-tech dependent factors, such as the shape and complexity of the aggregate trees, conversion rates, the presence of false positives, and handling of conversion values, to name a few. However, we have found significant improvements in Google Ads conversion data by leveraging the configurable components of the ARA.

## Concluding Remarks
Our goal with this blog post is to showcase the power of intelligent configuration of the ARA by focusing on customized MPC limits and sensitivity budgets. While these configurable levers can have a high impact on data accuracy, we wish to emphasize that there are several other configurable settings in both the Aggregate Summary reports and the Event-level reports; we encourage other ad-techs to explore these settings and to think carefully about configuration (and potentially customization) in order to maximize data utility.

In the Aggregate Summary Reports specifically, there are four additional aspects we have not discussed in detail in this post that we wish to highlight for ad-techs. We mention these here, because we view them as important configuration components which can have substantial impact on the utility of aggregate data.

* **Aggregate keys and structure**. Throughout our blog posts, we have assumed a hierarchical key setup for Aggregate Summary Reports, but this is not required by the ARA: ad-techs are free to choose the aggregate key configuration which works best for them. Even if an ad-tech uses a hierarchical structure, they still must decide which keys to use in the hierarchy and where to place them. 
* **Batching strategy**. Aggregate Summary Reports are generated on a cadence which is defined by the ad-tech. Here, encrypted reports are first sent to the ad-tech and then a “batch” is sent to the [Aggregation Service](https://github.com/WICG/attribution-reporting-api/blob/main/AGGREGATION_SERVICE_TEE.md) where aggregation happens and noise is applied. It is possible, for instance, for an ad-tech to acquire daily aggregates, but they could also choose to acquire aggregates on a weekly basis (or any other cadence).
* **Destination domain settings**. In cases where multiple conversion domains are possible, the ARA provides [support](https://github.com/WICG/attribution-reporting-api/issues/549) for allowing multiple potential attributable domains.  
* **Conversion value handling**. We mainly discuss the output of interest in Aggregate Summary Reports. However, the ARA can also be used to understand aggregated conversion values in cases where conversions are associated with differences in value. There are several ways one could collect values using Aggregate Summary Reports but we do not cover those methods here.

On the Event-level reports, we also wish to point out that API improvements such as the recent [flexible event-level configurations](https://github.com/WICG/attribution-reporting-api/blob/main/flexible_event_config.md) increase the Event-level Report flexibility (and therefore configurability). While the focus of this post is on aggregate data, we note that intelligent configuration and customization of Event-level Reports that leverages this new flexibility can also be valuable.
