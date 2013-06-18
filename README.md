# Mortar Github Recommender

This Mortar project contains the scripts which generate recommendations for [gitrec.mortardata.com](http://gitrec.mortardata.com/). This readme details how the algorithm works; for a just a quick overview, see [this blog post](http://blog.mortardata.com/post/53294300530/gitrec-your-personalized-github-repo-recommender).

The recommendations are generated by three pigscripts, `github_extract_events.pig`, `github_gen_item_recs.pig`, and `github_gen_user_recs.pig`, which are called using a controlscript `github_recommender.py`.

To be more generic (and possibly reusable), the code uses the term "item" instead of "repo". Also, if there is a thing that looks like a function call at the beginning of a pig statement, i.e. "user_ids = AssignIntegerIds(...)", that is actually a macro in `macros/recommender.pig`.

## Generating A User-Repo Affinity Graph

`github_extract_events.pig` parses raw Github event logs from the [Github Archive](http://www.githubarchive.org/) into a "user-repo affinity" graph where each edge represents that a user U has some connection to a repo R.

We consider four kinds of signals:

- user watches repo
- user forks repo
- user submits a pull request to a repo
- user pushes a commit to a repo.

These signals each have three values assigned to them: one for "specific interest" (in our case, pushes and pull requests express this), one for "general interest" (watches and forks express this), and lastly a "graph score" (a combination of both that is used to make the actual graph).

We aggregate each of these scores for each unique user-repo pair. To avoid problems where a hyper-active user who makes many commits to a repo every data messing up the algorithm, we use a logistic scaling function to map each aggregate score to a value in [0, 1] representing how engaged a user is with the repo.

`github_extract_events.pig` also extracts the most recent metadata we have available for each repo. Combining the number of forks and stars fields from the metadata (popularity) with the sum of all the user-repo affinity scores for a given repo (activity) gives an overall "importance score" for each repo.

## Finding The Neighborhood Of Each Repo

`github_gen_item_recs.pig` finds the collection of repos that are most similar to each individual repo, which we call its "neighborhood". It outputs these neighborhoods for use by the subsequent script, but it also joins a copy of them to repo metadata to produce final recommendations for "repos similar to the given repo" (we use DynamoDB to store the recommendations, so the output has to be denormalized).

The first step in generating the neighborhoods is to reduce the user-repo affinities into a repo-repo similarity graph. Each user contributes to a link between repos A and B if they have interacted with both A and B (i.e. forked both). We sum up every users contribution to the A-B link to get a "link score". We use both the "link score" for A-B and the "importance score" for B to estimate the P(user interacts with A | user interacts with B), incorporating a Bayesian prior to guard against small sample sizes. The reciprocal of this probability is said to be the distance of A->B. Note that these distances are not symmetric, that is A->B =/= B->A. Using this distance metric "close" repos are similar, "far" repos are dissimilar.

The reason for choosing to estimate P(A | B) for the A->B edge instead of P(B | A) is a bit counter-intuitive, but this example should help explain: suppose P(twitter/bootstrap | random/repo) = 1, that is every user who likes random/repo also likes twitter/bootstrap. That means there should be random/repo -> twitter/bootstrap should a short distance; but twitter/bootstrap -> random/repo should definitely not be short, since the users they share in common are only a tiny percentage of all the users of twitter/bootstrap.

To get distances for repos that are not directly connected, we follow shortest paths on the repo-repo distance graph up to a maximum path length of 4.

## Choosing Recommendations For Each User

`github_gen_user_recs.pig` does the last step of the recommender pipeline, choosing which repos to recommend for each user for each of two categories, "specific interest" (similar to those you've contributed to) and "general interest" (similar to those you've looked at). Once it generates it recommendations, it joins them to repo metadata to product the final denormalized output.

Choosing recommendations is simple: consider all the repos in the neighborhoods of repos the user has interacted with in each category (specific/general), then assign an new user-neighbor affinity score based on

1. the original user-repo affinity score for the repo that led us to the neighbor (called the "reason")
2. the distance between reason repo and the neighbor repo
3. the importance score of the neighbor repo. Then take the top N neighbor repos by affinity for each user.

Some care is also taken to avoid the following bad recommendations:

1. repos the user owns
2. repos the user has already interacted with
3. for the "general interest" category, repos which were already recommended as part of the "specific interest" category
