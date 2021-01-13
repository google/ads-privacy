

# FLoC Clustering Algorithm: k Random Centers

Based on [Federated Learning of Cohorts](https://github.com/jkarlin/floc "FLoC") (FLoC) original proposal, instead of including a unique identifier per user (such as a cookie) in each bid request,  we assign each user to a large *cohort* of users with similar browsing histories, and include only cohort IDs in bid requests. A cohort ID reveals a user’s general interests, but since any single cohort ID is shared by many users, it cannot be easily used to track individual browsing behavior across sites.

We determine a user’s cohort ID by applying a [locality-sensitive hash function](https://en.wikipedia.org/wiki/Locality-sensitive_hashing) to a vector representing the user’s browsing history. There are many ways to encode a user’s browsing history as a vector ![formula](https://render.githubusercontent.com/render/math?math=x). For example, we can number all websites from ![formula](https://render.githubusercontent.com/render/math?math=1) to ![formula](https://render.githubusercontent.com/render/math?math=d), and then let ![formula](https://render.githubusercontent.com/render/math?math=x) be a ![formula](https://render.githubusercontent.com/render/math?math=d)-dimensional vector whose ![formula](https://render.githubusercontent.com/render/math?math=i)th coordinate is ![formula](https://render.githubusercontent.com/render/math?math=1) if the user visited the ![formula](https://render.githubusercontent.com/render/math?math=i)th website and ![formula](https://render.githubusercontent.com/render/math?math=0) otherwise.

*k-random centers* is a simple locality-sensitive hash function. Given a ![formula](https://render.githubusercontent.com/render/math?math=d)-dimensional input vector ![formula](https://render.githubusercontent.com/render/math?math=x), the hash value ![formula](https://render.githubusercontent.com/render/math?math=H(x)) is determined as follows:

1. Choose ![formula](https://render.githubusercontent.com/render/math?math=k) random points (‘centers’) in ![formula](https://render.githubusercontent.com/render/math?math=d)-dimensional space, and number them ![formula](https://render.githubusercontent.com/render/math?math=1) to ![formula](https://render.githubusercontent.com/render/math?math=k).
2. Let ![formula](https://render.githubusercontent.com/render/math?math=H(x)) be the index of the center closest to x according to [cosine similarity](https://en.wikipedia.org/wiki/Cosine_similarity)

We let ![formula](https://render.githubusercontent.com/render/math?math=H(x)) be the cohort ID of a user with history vector ![formula](https://render.githubusercontent.com/render/math?math=x). For comparison with other algorithms, note that since the cohort ID is the index of a center, the number of bits required to encode the ID is ![formula](https://render.githubusercontent.com/render/math?math=ceil(log_{2}(k))).

See figure below for an illustration of this hash function.

![alt text](https://github.com/google/ads-privacy/blob/master/proposals/FLoC/k-random%20centers.svg "k-random-centers")

Each input vector is assigned to its closest center, and the hash value is the index of that center. In this example, vector ![formula](https://render.githubusercontent.com/render/math?math=x_1) is assigned to center 1, and vectors ![formula](https://render.githubusercontent.com/render/math?math=x_2) and ![formula](https://render.githubusercontent.com/render/math?math=x_3) are assigned to center 2.

Another way to understand the k-random centers hash function is to note that it is essentially equivalent to the first iteration of the standard [k-means clustering algorithm](https://en.wikipedia.org/wiki/K-means_clustering#Standard_algorithm_(na%C3%AFve_k-means)).

A key advantage of k-random centers (and any locality-sensitive hash function) is that it can be implemented in a fully distributed manner without the need for a central server that stores private user data. If all users use the same pseudo-random seed to generate the random centers, then each user’s cohort ID can be calculated independently by each user without any communication or coordination among the users, and yet similar browsing histories will nonetheless be mapped to the same cohort ID.

The main downsides of a fully distributed cohort assignment algorithm are that (1) within-cohort similarity is weaker than for a more centralized algorithm, and (2) a minimum cohort size cannot be enforced. With respect to the latter, we have observed that in practice cohort sizes are roughly uniformly distributed, with the number of users per cohort decreasing as the number of centers increases.
