# Pagerank Project

In this project, you will create a simple search engine for the website <https://www.lawfareblog.com>.
This website provides legal analysis on US national security issues.

**Due date:** Sunday, 18 September at midnight

**Computation:**
This project has low computational requirements.
You are not required to complete it on the lambda server (although you are welcome to if you'd like).

## Background

**Data:**

The `data` folder contains two files that store example "web graphs".
The file `small.csv.gz` contains the example graph from the *Deeper Inside Pagerank* paper.
This is a small graph, so we can manually inspect the contents of this file with the following command:
```
$ zcat data/small.csv.gz
source,target
1,2
1,3
3,1
3,2
3,5
4,5
4,6
5,6
5,4
6,4
```

> **Recall:**
> The `cat` terminal command outputs the contents of a file to stdout, and the `zcat` command first decompressed a gzipped file and then outputs the decompressed contents.

As you can see, the graph is stored as a CSV file.
The first line is a header,
and each subsequent line stores a single edge in the graph.
The first column contains the source node of the edge and the second column the target node.
The file is assumed to be sorted alphabetically.

The second data file `lawfareblog.csv.gz` contains the link structure for the lawfare blog.
Let's take a look at the first 10 of these lines:
```
$ zcat data/lawfareblog.csv.gz | head
source,target
www.lawfareblog.com/,www.lawfareblog.com/topic/interrogation
www.lawfareblog.com/,www.lawfareblog.com/upcoming-events
www.lawfareblog.com/,www.lawfareblog.com/
www.lawfareblog.com/,www.lawfareblog.com/our-comments-policy
www.lawfareblog.com/,www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
www.lawfareblog.com/,www.lawfareblog.com/topic/lawfare-research-paper-series
www.lawfareblog.com/,www.lawfareblog.com/topic/book-reviews
www.lawfareblog.com/,www.lawfareblog.com/documents-related-mueller-investigation
www.lawfareblog.com/,www.lawfareblog.com/topic/international-law-loac
```
You can see that in this file, the node names are URLs.
Semantically, each line corresponds to an HTML `<a>` tag that is contained in the source webpage and links to the target webpage.

We can use the following command to count the total number of links in the file:
```
$ zcat data/lawfareblog.csv.gz | wc -l
1610789
```
Since every link corresponds to a non-zero entry in the `P` matrix,
this is also the value of `nnz(P)`.
(Technically, we should subtract 1 from this value since the `wc -l` command also counts the header line, not just the data lines.)

To get the dimensions of `P`, we need to count the total number of nodes in the graph.
The following command achieves this by: decompressing the file, extracting the first column, removing all duplicate lines, then counting the results.
```
$ zcat data/lawfareblog.csv.gz | cut -f1 -d, | uniq | wc -l
25761
```
This matrix is large enough that computing matrix products for dense matrices takes several minutes on a single CPU.
Fortunately, however, the matrix is very sparse.
The following python code computes the fraction of entries in the matrix with non-zero values:
```
>>> 1610788 / (25760**2)
0.0024274297384360172
```
Thus, by using sparse matrix operations, we will be able to speed up the code significantly.

**Code:**

The `pagerank.py` file contains code for loading the graph CSV files and searching through their nodes for key phrases.
For example, you can perform a search for all nodes (i.e. urls) that mention the string `corona` with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --search_query=corona
```

> **NOTE:**
> It will take about 10 seconds to load and parse the data files.
> All the other computation happens essentially instantly.

Currently, the pagerank of the nodes is not currently being calculated correctly, and so the webpages are returned in an arbitrary order.
Your task in this assignment will be to fix these calculations in order to have the most important results (i.e. highest pagerank results) returned first.

## Task 1: the power method

Implement the `WebGraph.power_method` function in `pagerank.py` for computing the pagerank vector by fixing the `FIXME` annotation.

**Part 1:**

To check that your implementation is working,
you should run the program on the `data/small.csv.gz` graph.
For my implementation, I get the following output.
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=2.5629e-01
DEBUG:root:i=1 residual=1.1841e-01
DEBUG:root:i=2 residual=7.0701e-02
DEBUG:root:i=3 residual=3.1815e-02
DEBUG:root:i=4 residual=2.0497e-02
DEBUG:root:i=5 residual=1.0108e-02
DEBUG:root:i=6 residual=6.3716e-03
DEBUG:root:i=7 residual=3.4228e-03
DEBUG:root:i=8 residual=2.0879e-03
DEBUG:root:i=9 residual=1.1750e-03
DEBUG:root:i=10 residual=7.0131e-04
DEBUG:root:i=11 residual=4.0321e-04
DEBUG:root:i=12 residual=2.3800e-04
DEBUG:root:i=13 residual=1.3812e-04
DEBUG:root:i=14 residual=8.1083e-05
DEBUG:root:i=15 residual=4.7251e-05
DEBUG:root:i=16 residual=2.7704e-05
DEBUG:root:i=17 residual=1.6164e-05
DEBUG:root:i=18 residual=9.4778e-06
DEBUG:root:i=19 residual=5.5066e-06
DEBUG:root:i=20 residual=3.2042e-06
DEBUG:root:i=21 residual=1.8612e-06
DEBUG:root:i=22 residual=1.1283e-06
DEBUG:root:i=23 residual=6.1907e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1
```
Yours likely won't be identical (due to weird floating point issues), but it should be similar.
In particular, the ranking of the nodes/urls should be the same order.

> **NOTE:**
> The `--verbose` flag causes all of the lines beginning with `DEBUG` to be printed.
> By default, only lines beginning with `INFO` are printed.

**Part 2:**

The `pagerank.py` file has an option `--search_query`, which takes a string as a parameter.
If this argument is used, then the program returns all nodes that match the query string sorted according to their pagerank.
Essentially, this gives us the most important pages related to our query.

Again, you may not get the exact same results as me,
but you should get similar results to the examples I've shown below.
Verify that you do in fact get similar results.

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9224e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0390e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9153e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7041e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6256e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5046e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3620e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1248e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0187e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 pagerank=5.7826e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2338e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1297e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6599e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5934e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3071e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0935e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7591e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4509e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4484e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 pagerank=4.5746e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4174e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6928e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9391e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5452e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5357e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4221e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
```

**Part 3:**

The webgraph of lawfareblog.com (i.e. the `P` matrix) naturally contains a lot of structure.
For example, essentially all pages on the domain have links to the root page <https://lawfareblog.com/> and other "non-article" pages like <https://www.lawfareblog.com/topics> and <https://www.lawfareblog.com/subscribe-lawfare>.
These pages therefore have a large pagerank.
We can get a list of the pages with the largest pagerank by running

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
```

Most of these pages are not very interesting, however, because they are not articles,
and usually when we are performing a web search, we only want articles.

This raises the question: How can we find the most important articles filtering out the non-article pages?
The answer is to modify the `P` matrix by removing all links to non-article pages.

This raises another question: How do we know if a link is a non-article page?
Unfortunately, this is a hard question to answer with 100% accuracy,
but there are many methods that get us most of the way there.
One easy to implement method is to compute what's called the "in-link ratio" of each node (i.e. the total number of edges with the node as a target divided by the total number of nodes),
and then remove nodes from the search results with too-high of a ratio.
The intuition is that non-article pages often appear in the menu of a webpage, and so have links from almost all of the other webpages;
but article-webpages are unlikely to appear on a menu and so will only have a small number of links from other webpages.
The `--filter_ratio` parameter causes the code to remove all pages that have an in-link ratio larger than the provided value.

Using this option, we can estimate the most important articles on the domain with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
```
Notice that the urls in this list look much more like articles than the urls in the previous list.

When Google calculates their `P` matrix for the web,
they use a similar (but much more complicated) process to modify the `P` matrix in order to reduce spam results.
The exact formula they use is a jealously guarded secret that they update continuously.

In the case above, notice that we have accidentally removed the blog's most popular article (<www.lawfareblog.com/snowden-revelations>).
The blog editors believed that Snowden's revelations about NSA spying are so important that they directly put a link to the article on the menu.
So every single webpage in the domain links to the Snowden article,
and our "anti-spam" `--filter-ratio` argument removed this article from the list.
In general, it is a challenging open problem to remove spam from pagerank results,
and all current solutions rely on careful human tuning and still have lots of false positives and false negatives.

**Part 4:**

Recall from the reading that the runtime of pagerank depends heavily on the eigengap of the `\bar\bar P` matrix,
and that this eigengap is bounded by the alpha parameter.

Run the following four commands:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
```
You should notice that the last command takes considerably more iterations to compute the pagerank vector.
(My code takes 685 iterations for this call, and about 10 iterations for all the others.)

This raises the question: Why does the second command (with the `--alpha` option but without the `--filter_ratio`) option not take a long time to run?
The answer is that the `P` graph for <https://www.lawfareblog.com> naturally has a large eigengap and so is fast to compute for all alpha values,
but the modified graph does not have a large eigengap and so requires a small alpha for fast convergence.

Changing the value of alpha also gives us very different pagerank rankings.
For example, 
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
INFO:root:rank=0 pagerank=7.0149e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0149e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0552e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1755e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2040e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6027e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6026e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6023e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
```

Which of these rankings is better is entirely subjective,
and the only way to know if you have the "best" alpha for your application is to try several variations and see what is best.
If large alphas are good for your application, you can see that there is a trade-off between quality answers and algorithmic runtime.
We'll be exploring this trade-off more formally in class over the rest of the semester.

## Task 2: the personalization vector

The most interesting applications of pagerank involve the personalization vector.
Implement the `WebGraph.make_personalization_vector` function so that it outputs a personalization vector tuned for the input query.
The pseudocode for the function is:
```
for each index in the personalization vector:
    get the url for the index (see the _index_to_url function)
    check if the url satisfies the input query (see the url_satisfies_query function)
    if so, set the corresponding index to one
normalize the vector
```

**Part 1:**

The command line argument `--personalization_vector_query` will use the function you created above to augment your search with a custom personalization vector.
If you've implemented the function correctly,
you should get results similar to:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 pagerank=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 pagerank=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
```

Notice that these results are significantly different than when using the `--search_query` option:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --search_query='corona'
INFO:root:rank=0 pagerank=8.1320e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=1 pagerank=7.7908e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=2 pagerank=5.2262e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=3 pagerank=3.9584e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=4 pagerank=3.8114e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=5 pagerank=3.3973e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=6 pagerank=3.3633e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus
INFO:root:rank=7 pagerank=3.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=8 pagerank=3.2160e-03 url=www.lawfareblog.com/congress-needs-coronavirus-failsafe-its-too-late
INFO:root:rank=9 pagerank=3.1036e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
```

Which results are better?
Again, that depends on what you mean by "better."
With the `--personalization_vector_query` option,
a webpage is important only if other coronavirus webpages also think it's important;
with the `--search_query` option,
a webpage is important if any other webpage thinks it's important.
You'll notice that in the later example, many of the webpages are about Congressional proceedings related to the coronavirus.
From a strictly coronavirus perspective, these are not very important webpages.
But in the broader context of national security, these are very important webpages.

Google engineers spend TONs of time fine-tuning their pagerank personalization vectors to remove spam webpages.
Exactly how they do this is another one of their secrets that they don't publicly talk about.

**Part 2:**

Another use of the `--personalization_vector_query` option is that we can find out what webpages are related to the coronavirus but don't directly mention the coronavirus.
This can be used to map out what types of topics are similar to the coronavirus.

For example, the following query ranks all webpages by their `corona` importance,
but removes webpages mentioning `corona` from the results.
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
```
You can see that there are many urls about concepts that are obviously related like "covid", "clinical trials", and "quarantine",
but this algorithm also finds articles about Chinese propaganda and Trump's policy decisions.
Both of these articles are highly relevant to coronavirus discussions,
but a simple keyword search for corona or related terms would not find these articles.

<--
**Part 3:**

Select another topic related to national security.
You should experiment with a national security topic other than the coronavirus.
For example, find out what articles are important to the `iran` topic but do not contain the word `iran`.
Your goal should be to discover what topics that www.lawfareblog.com considers to be related to the national security topic you choose.
-->

## Submission

1. Create a new repo on github (not a fork of this repo).

1. Run the following commands, and paste their output into the code blocks below.
   
   Task 1, part 1:
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=0.2562914192676544
DEBUG:root:i=1 residual=0.11841222643852234
DEBUG:root:i=2 residual=0.07070135325193405
DEBUG:root:i=3 residual=0.03181542828679085
DEBUG:root:i=4 residual=0.02049657516181469
DEBUG:root:i=5 residual=0.010108369402587414
DEBUG:root:i=6 residual=0.006371586117893457
DEBUG:root:i=7 residual=0.003422787180170417
DEBUG:root:i=8 residual=0.002087953267619014
DEBUG:root:i=9 residual=0.0011749608675017953
DEBUG:root:i=10 residual=0.0007013111608102918
DEBUG:root:i=11 residual=0.00040320667903870344
DEBUG:root:i=12 residual=0.0002379981888225302
DEBUG:root:i=13 residual=0.0001381236652377993
DEBUG:root:i=14 residual=8.10831697890535e-05
DEBUG:root:i=15 residual=4.7250770876416937e-05
DEBUG:root:i=16 residual=2.7704092644853517e-05
DEBUG:root:i=17 residual=1.616420195205137e-05
DEBUG:root:i=18 residual=9.47775970416842e-06
DEBUG:root:i=19 residual=5.506619345396757e-06
DEBUG:root:i=20 residual=3.204200083928299e-06
DEBUG:root:i=21 residual=1.8612140593177173e-06
DEBUG:root:i=22 residual=1.1282648983979016e-06
DEBUG:root:i=23 residual=6.190710450937331e-07
INFO:root:rank=0 ranking=6.6270e-01 url=4
INFO:root:rank=1 ranking=5.2179e-01 url=6
INFO:root:rank=2 ranking=4.1434e-01 url=5
INFO:root:rank=3 ranking=2.3175e-01 url=2
INFO:root:rank=4 ranking=1.8590e-01 url=3
INFO:root:rank=5 ranking=1.6917e-01 url=1
   ```

   Task 1, part 2:
   ```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
 INFO:root:rank=0 ranking=0.0000e+00 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
 INFO:root:rank=1 ranking=0.0000e+00 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
 INFO:root:rank=2 ranking=0.0000e+00 url=www.lawfareblog.com/britains-coronavirus-response
 INFO:root:rank=3 ranking=0.0000e+00 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
 INFO:root:rank=4 ranking=0.0000e+00 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
 INFO:root:rank=5 ranking=0.0000e+00 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
 INFO:root:rank=6 ranking=0.0000e+00 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
 INFO:root:rank=7 ranking=0.0000e+00 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
 INFO:root:rank=8 ranking=0.0000e+00 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
 INFO:root:rank=9 ranking=0.0000e+00 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
 INFO:root:rank=0 ranking=3.0563e-16 url=www.lawfareblog.com/will-we-ever-learn-what-bob-mueller-knows
 INFO:root:rank=1 ranking=2.9258e-16 url=www.lawfareblog.com/tom-malinowski-responds-autonomous-lethal-systems
 INFO:root:rank=2 ranking=2.8979e-16 url=www.lawfareblog.com/tom-malinowski-ups-game-lawfares-discussion-killer-robots
 INFO:root:rank=3 ranking=2.8825e-16 url=www.lawfareblog.com/original-understanding-malinowski
 INFO:root:rank=4 ranking=2.8702e-16 url=www.lawfareblog.com/tom-malinowski-responds-lethal-autonomous-systems-part-ii
 INFO:root:rank=5 ranking=2.8326e-16 url=www.lawfareblog.com/national-security-law-podcast-everyone-knows-it-saudi-arabia
 INFO:root:rank=6 ranking=2.8304e-16 url=www.lawfareblog.com/human-rights-watch-campaign-killer-robots-and-tom-malinowskis-response
 INFO:root:rank=7 ranking=2.8130e-16 url=www.lawfareblog.com/clarification-tom-malinowski
 INFO:root:rank=8 ranking=2.7820e-16 url=www.lawfareblog.com/what-mueller-knows-ensuring-special-counsel-report-worst-case-scenario
 INFO:root:rank=9 ranking=2.7741e-16 url=www.lawfareblog.com/june-11-session-8-what-habeas-counsel-knows


$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
 INFO:root:rank=0 ranking=2.8981e-07 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
 INFO:root:rank=1 ranking=2.8366e-07 url=www.lawfareblog.com/and-then-there-was-one-tehran-still-has-one-iranian-american-behind-bars
 INFO:root:rank=2 ranking=2.6836e-07 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
 INFO:root:rank=3 ranking=1.9256e-07 url=www.lawfareblog.com/trump-wants-bigger-better-deal-iran-what-does-tehran-want
 INFO:root:rank=4 ranking=1.9230e-07 url=www.lawfareblog.com/iranian-terrorism-victims-ask-court-block-release-iranian-assets-under-nuclear-deal
 INFO:root:rank=5 ranking=1.8720e-07 url=www.lawfareblog.com/afghanistan-another-victory-tehran
 INFO:root:rank=6 ranking=1.4129e-07 url=www.lawfareblog.com/iranian-protesters-strike-heart-regimes-revolutionary-legitimacy
 INFO:root:rank=7 ranking=1.0117e-07 url=www.lawfareblog.com/what-has-iran-done-now-primer-recent-iranian-missile-tests-and-sanctions
 INFO:root:rank=8 ranking=1.0044e-07 url=www.lawfareblog.com/two-further-notes-npt-and-iranian-legal-arguments
 INFO:root:rank=9 ranking=9.9820e-08 url=www.lawfareblog.com/iranian-missile-launch-and-gray-ladys-confusion
   ```

   Task 1, part 3:
   ```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 ranking=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 ranking=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 ranking=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 ranking=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 ranking=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 ranking=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 ranking=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 ranking=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 ranking=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 ranking=2.8741e-01 url=www.lawfareblog.com/our-comments-policy


$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 ranking=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 ranking=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 ranking=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 ranking=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 ranking=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 ranking=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 ranking=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 ranking=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 ranking=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 ranking=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

   ```

   Task 1, part 4:
   ```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.3793821334838867
DEBUG:root:i=1 residual=0.11642514914274216
DEBUG:root:i=2 residual=0.07495073974132538
DEBUG:root:i=3 residual=0.031712669879198074
DEBUG:root:i=4 residual=0.01746140606701374
DEBUG:root:i=5 residual=0.008529541082680225
DEBUG:root:i=6 residual=0.004439213313162327
DEBUG:root:i=7 residual=0.002238804241642356
DEBUG:root:i=8 residual=0.0011464771814644337
DEBUG:root:i=9 residual=0.0005798120982944965
DEBUG:root:i=10 residual=0.00029221168369986117
DEBUG:root:i=11 residual=0.0001456011668778956
DEBUG:root:i=12 residual=7.149929297156632e-05
DEBUG:root:i=13 residual=3.434011887293309e-05
DEBUG:root:i=14 residual=1.5638997865607962e-05
DEBUG:root:i=15 residual=6.3710658650961705e-06
DEBUG:root:i=16 residual=2.893735199904768e-06
DEBUG:root:i=17 residual=1.356311031486257e-06
DEBUG:root:i=18 residual=4.3140809680153325e-07
INFO:root:rank=0 ranking=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 ranking=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 ranking=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 ranking=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 ranking=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 ranking=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 ranking=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 ranking=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 ranking=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 ranking=2.8741e-01 url=www.lawfareblog.com/our-comments-policy


$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.3845856189727783
DEBUG:root:i=1 residual=0.07088156044483185
DEBUG:root:i=2 residual=0.01882227510213852
DEBUG:root:i=3 residual=0.006958263460546732
DEBUG:root:i=4 residual=0.0027358192019164562
DEBUG:root:i=5 residual=0.0010345557238906622
DEBUG:root:i=6 residual=0.0003774628567043692
DEBUG:root:i=7 residual=0.00013533400488086045
DEBUG:root:i=8 residual=4.8224086640402675e-05
DEBUG:root:i=9 residual=1.7172402294818312e-05
DEBUG:root:i=10 residual=6.116189069871325e-06
DEBUG:root:i=11 residual=2.172585482185241e-06
DEBUG:root:i=12 residual=7.764264751131122e-07
INFO:root:rank=0 ranking=2.8859e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 ranking=2.8859e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 ranking=2.8859e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 ranking=2.8859e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 ranking=2.8859e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 ranking=2.8859e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 ranking=2.8859e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 ranking=2.8859e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 ranking=2.8859e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 ranking=2.8859e-01 url=www.lawfareblog.com/our-comments-policy

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.2609827518463135
DEBUG:root:i=1 residual=0.4985857307910919
DEBUG:root:i=2 residual=0.13420067727565765
DEBUG:root:i=3 residual=0.0692317932844162
DEBUG:root:i=4 residual=0.023411503061652184
DEBUG:root:i=5 residual=0.010188627988100052
DEBUG:root:i=6 residual=0.004910625983029604
DEBUG:root:i=7 residual=0.002279868582263589
DEBUG:root:i=8 residual=0.001073912950232625
DEBUG:root:i=9 residual=0.0005249498644843698
DEBUG:root:i=10 residual=0.0002697036834433675
DEBUG:root:i=11 residual=0.00014575463137589395
DEBUG:root:i=12 residual=8.240594615926966e-05
DEBUG:root:i=13 residual=4.819399691768922e-05
DEBUG:root:i=14 residual=2.881712862290442e-05
DEBUG:root:i=15 residual=1.7420363292330876e-05
DEBUG:root:i=16 residual=1.0547823876549955e-05
DEBUG:root:i=17 residual=6.381446837622207e-06
DEBUG:root:i=18 residual=3.836741143459221e-06
DEBUG:root:i=19 residual=2.299691686857841e-06
DEBUG:root:i=20 residual=1.3633346043206984e-06
DEBUG:root:i=21 residual=8.096001806734421e-07
INFO:root:rank=0 ranking=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 ranking=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 ranking=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 ranking=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 ranking=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 ranking=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 ranking=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 ranking=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 ranking=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 ranking=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

 $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
  DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=1.2827345132827759
DEBUG:root:i=1 residual=0.5695679783821106
DEBUG:root:i=2 residual=0.38298743963241577
DEBUG:root:i=3 residual=0.217390775680542
DEBUG:root:i=4 residual=0.14045001566410065
DEBUG:root:i=5 residual=0.10851321369409561
DEBUG:root:i=6 residual=0.09284131973981857
DEBUG:root:i=7 residual=0.08225570619106293
DEBUG:root:i=8 residual=0.07338865846395493
DEBUG:root:i=9 residual=0.06561221927404404
DEBUG:root:i=10 residual=0.05909644439816475
DEBUG:root:i=11 residual=0.0541754849255085
DEBUG:root:i=12 residual=0.05111688748002052
DEBUG:root:i=13 residual=0.04999382048845291
DEBUG:root:i=14 residual=0.050609007477760315
DEBUG:root:i=15 residual=0.05252634361386299
DEBUG:root:i=16 residual=0.05518888309597969
DEBUG:root:i=17 residual=0.058038514107465744
DEBUG:root:i=18 residual=0.06059221550822258
DEBUG:root:i=19 residual=0.0624784417450428
DEBUG:root:i=20 residual=0.06345317512750626
DEBUG:root:i=21 residual=0.06340520083904266
DEBUG:root:i=22 residual=0.06234564632177353
DEBUG:root:i=23 residual=0.06038397178053856
DEBUG:root:i=24 residual=0.05769401788711548
DEBUG:root:i=25 residual=0.05447974428534508
DEBUG:root:i=26 residual=0.05094275623559952
DEBUG:root:i=27 residual=0.047261033207178116
DEBUG:root:i=28 residual=0.043578553944826126
DEBUG:root:i=29 residual=0.04000163823366165
DEBUG:root:i=30 residual=0.03660249710083008
DEBUG:root:i=31 residual=0.03342431038618088
DEBUG:root:i=32 residual=0.030489446595311165
DEBUG:root:i=33 residual=0.02780323661863804
DEBUG:root:i=34 residual=0.0253607165068388
DEBUG:root:i=35 residual=0.023149998858571053
DEBUG:root:i=36 residual=0.02115532197058201
DEBUG:root:i=37 residual=0.019359193742275238
DEBUG:root:i=38 residual=0.017743488773703575
DEBUG:root:i=39 residual=0.016290778294205666
DEBUG:root:i=40 residual=0.014984299428761005
DEBUG:root:i=41 residual=0.013808553107082844
DEBUG:root:i=42 residual=0.012749727815389633
DEBUG:root:i=43 residual=0.011794866062700748
DEBUG:root:i=44 residual=0.010932844132184982
DEBUG:root:i=45 residual=0.010153336450457573
DEBUG:root:i=46 residual=0.009447365999221802
DEBUG:root:i=47 residual=0.008807015605270863
DEBUG:root:i=48 residual=0.00822509080171585
DEBUG:root:i=49 residual=0.007695485837757587
DEBUG:root:i=50 residual=0.007212589494884014
DEBUG:root:i=51 residual=0.006771544925868511
DEBUG:root:i=52 residual=0.006367984227836132
DEBUG:root:i=53 residual=0.005998124834150076
DEBUG:root:i=54 residual=0.005658615846186876
DEBUG:root:i=55 residual=0.005346364341676235
DEBUG:root:i=56 residual=0.0050587961450219154
DEBUG:root:i=57 residual=0.004793431609869003
DEBUG:root:i=58 residual=0.004548263270407915
DEBUG:root:i=59 residual=0.004321323707699776
DEBUG:root:i=60 residual=0.004110999405384064
DEBUG:root:i=61 residual=0.0039157383143901825
DEBUG:root:i=62 residual=0.003734213998541236
DEBUG:root:i=63 residual=0.0035652003716677427
DEBUG:root:i=64 residual=0.003407652024179697
DEBUG:root:i=65 residual=0.0032605663873255253
DEBUG:root:i=66 residual=0.003123007481917739
DEBUG:root:i=67 residual=0.002994283102452755
DEBUG:root:i=68 residual=0.002873630030080676
DEBUG:root:i=69 residual=0.002760406117886305
DEBUG:root:i=70 residual=0.002654000651091337
DEBUG:root:i=71 residual=0.0025539309717714787
DEBUG:root:i=72 residual=0.0024596620351076126
DEBUG:root:i=73 residual=0.00237078033387661
DEBUG:root:i=74 residual=0.002286891918629408
DEBUG:root:i=75 residual=0.0022076331079006195
DEBUG:root:i=76 residual=0.00213263975456357
DEBUG:root:i=77 residual=0.002061642473563552
DEBUG:root:i=78 residual=0.001994357444345951
DEBUG:root:i=79 residual=0.0019305212190374732
DEBUG:root:i=80 residual=0.001869876985438168
DEBUG:root:i=81 residual=0.0018122534966096282
DEBUG:root:i=82 residual=0.0017574327066540718
DEBUG:root:i=83 residual=0.0017052164766937494
DEBUG:root:i=84 residual=0.001655459520407021
DEBUG:root:i=85 residual=0.0016080194618552923
DEBUG:root:i=86 residual=0.001562709454447031
DEBUG:root:i=87 residual=0.0015194363659247756
DEBUG:root:i=88 residual=0.0014780613128095865
DEBUG:root:i=89 residual=0.0014384721871465445
DEBUG:root:i=90 residual=0.0014005588600412011
DEBUG:root:i=91 residual=0.0013642325066030025
DEBUG:root:i=92 residual=0.0013293962692841887
DEBUG:root:i=93 residual=0.0012959676096215844
DEBUG:root:i=94 residual=0.001263874932192266
DEBUG:root:i=95 residual=0.001233043265528977
DEBUG:root:i=96 residual=0.0012033991515636444
DEBUG:root:i=97 residual=0.001174874952994287
DEBUG:root:i=98 residual=0.001147418050095439
DEBUG:root:i=99 residual=0.0011209729127585888
DEBUG:root:i=100 residual=0.0010954836616292596
DEBUG:root:i=101 residual=0.0010709082707762718
DEBUG:root:i=102 residual=0.0010471954010426998
DEBUG:root:i=103 residual=0.0010243069846183062
DEBUG:root:i=104 residual=0.0010021915659308434
DEBUG:root:i=105 residual=0.0009808208560571074
DEBUG:root:i=106 residual=0.0009601617348380387
DEBUG:root:i=107 residual=0.0009401699644513428
DEBUG:root:i=108 residual=0.000920825928915292
DEBUG:root:i=109 residual=0.0009020984871312976
DEBUG:root:i=110 residual=0.0008839552174322307
DEBUG:root:i=111 residual=0.0008663738844916224
DEBUG:root:i=112 residual=0.000849321426358074
DEBUG:root:i=113 residual=0.0008327725809067488
DEBUG:root:i=114 residual=0.000816724612377584
DEBUG:root:i=115 residual=0.0008011538302525878
DEBUG:root:i=116 residual=0.0007860134355723858
DEBUG:root:i=117 residual=0.0007713024388067424
DEBUG:root:i=118 residual=0.0007570094894617796
DEBUG:root:i=119 residual=0.0007431223057210445
DEBUG:root:i=120 residual=0.0007295985706150532
DEBUG:root:i=121 residual=0.0007164514972828329
DEBUG:root:i=122 residual=0.0007036479655653238
DEBUG:root:i=123 residual=0.0006911890814080834
DEBUG:root:i=124 residual=0.0006790345069020987
DEBUG:root:i=125 residual=0.0006672081653960049
DEBUG:root:i=126 residual=0.0006556775770150125
DEBUG:root:i=127 residual=0.0006444310420192778
DEBUG:root:i=128 residual=0.0006334696081466973
DEBUG:root:i=129 residual=0.0006227672565728426
DEBUG:root:i=130 residual=0.000612326548434794
DEBUG:root:i=131 residual=0.0006021408480592072
DEBUG:root:i=132 residual=0.0005921921692788601
DEBUG:root:i=133 residual=0.0005824733525514603
DEBUG:root:i=134 residual=0.0005729837575927377
DEBUG:root:i=135 residual=0.0005637063295580447
DEBUG:root:i=136 residual=0.0005546404863707721
DEBUG:root:i=137 residual=0.0005457792431116104
DEBUG:root:i=138 residual=0.0005371191655285656
DEBUG:root:i=139 residual=0.0005286399973556399
DEBUG:root:i=140 residual=0.0005203532055020332
DEBUG:root:i=141 residual=0.0005122333532199264
DEBUG:root:i=142 residual=0.0005042988923378289
DEBUG:root:i=143 residual=0.0004965270054526627
DEBUG:root:i=144 residual=0.0004889105912297964
DEBUG:root:i=145 residual=0.000481454364489764
DEBUG:root:i=146 residual=0.0004741560260299593
DEBUG:root:i=147 residual=0.00046700387611053884
DEBUG:root:i=148 residual=0.0004599929670803249
DEBUG:root:i=149 residual=0.00045312708243727684
DEBUG:root:i=150 residual=0.00044639076804742217
DEBUG:root:i=151 residual=0.0004397900775074959
DEBUG:root:i=152 residual=0.0004333139513619244
DEBUG:root:i=153 residual=0.000426971324486658
DEBUG:root:i=154 residual=0.00042074164957739413
DEBUG:root:i=155 residual=0.00041463173693045974
DEBUG:root:i=156 residual=0.0004086458939127624
DEBUG:root:i=157 residual=0.00040275888750329614
DEBUG:root:i=158 residual=0.00039698294131085277
DEBUG:root:i=159 residual=0.0003913175023626536
DEBUG:root:i=160 residual=0.00038575223879888654
DEBUG:root:i=161 residual=0.0003802933788392693
DEBUG:root:i=162 residual=0.00037492826231755316
DEBUG:root:i=163 residual=0.0003696566272992641
DEBUG:root:i=164 residual=0.0003644873504526913
DEBUG:root:i=165 residual=0.0003593998262658715
DEBUG:root:i=166 residual=0.00035440069041214883
DEBUG:root:i=167 residual=0.0003494936681818217
DEBUG:root:i=168 residual=0.0003446697082836181
DEBUG:root:i=169 residual=0.0003399324486963451
DEBUG:root:i=170 residual=0.0003352703934069723
DEBUG:root:i=171 residual=0.0003306936996523291
DEBUG:root:i=172 residual=0.00032618766999803483
DEBUG:root:i=173 residual=0.0003217585035599768
DEBUG:root:i=174 residual=0.00031740195117890835
DEBUG:root:i=175 residual=0.0003131211851723492
DEBUG:root:i=176 residual=0.00030891320784576237
DEBUG:root:i=177 residual=0.0003047646314371377
DEBUG:root:i=178 residual=0.00030069643980823457
DEBUG:root:i=179 residual=0.00029668427305296063
DEBUG:root:i=180 residual=0.0002927418681792915
DEBUG:root:i=181 residual=0.0002888584276661277
DEBUG:root:i=182 residual=0.00028504044166766107
DEBUG:root:i=183 residual=0.00028127714176662266
DEBUG:root:i=184 residual=0.000277580926194787
DEBUG:root:i=185 residual=0.00027393741765990853
DEBUG:root:i=186 residual=0.00027035665698349476
DEBUG:root:i=187 residual=0.000266824645223096
DEBUG:root:i=188 residual=0.0002633550902828574
DEBUG:root:i=189 residual=0.00025993259623646736
DEBUG:root:i=190 residual=0.00025656071375124156
DEBUG:root:i=191 residual=0.00025324520538561046
DEBUG:root:i=192 residual=0.0002499794354662299
DEBUG:root:i=193 residual=0.00024676264729350805
DEBUG:root:i=194 residual=0.0002435936767142266
DEBUG:root:i=195 residual=0.00024047278566285968
DEBUG:root:i=196 residual=0.00023740129836369306
DEBUG:root:i=197 residual=0.0002343689266126603
DEBUG:root:i=198 residual=0.00023138684628065675
DEBUG:root:i=199 residual=0.00022844642808195204
DEBUG:root:i=200 residual=0.00022554966562893242
DEBUG:root:i=201 residual=0.00022269862529356033
DEBUG:root:i=202 residual=0.000219883571844548
DEBUG:root:i=203 residual=0.0002171109663322568
DEBUG:root:i=204 residual=0.00021437890245579183
DEBUG:root:i=205 residual=0.00021168890816625208
DEBUG:root:i=206 residual=0.00020903618133161217
DEBUG:root:i=207 residual=0.00020642195886466652
DEBUG:root:i=208 residual=0.0002038449893007055
DEBUG:root:i=209 residual=0.0002013043558690697
DEBUG:root:i=210 residual=0.0001987970608752221
DEBUG:root:i=211 residual=0.00019632575276773423
DEBUG:root:i=212 residual=0.0001938901114044711
DEBUG:root:i=213 residual=0.00019149135914631188
DEBUG:root:i=214 residual=0.0001891234569484368
DEBUG:root:i=215 residual=0.00018678944616112858
DEBUG:root:i=216 residual=0.00018448995251674205
DEBUG:root:i=217 residual=0.00018221698701381683
DEBUG:root:i=218 residual=0.00017998080875258893
DEBUG:root:i=219 residual=0.00017777546599972993
DEBUG:root:i=220 residual=0.00017559886327944696
DEBUG:root:i=221 residual=0.00017345183005090803
DEBUG:root:i=222 residual=0.00017133544315584004
DEBUG:root:i=223 residual=0.0001692441728664562
DEBUG:root:i=224 residual=0.00016718629922252148
DEBUG:root:i=225 residual=0.00016515354218427092
DEBUG:root:i=226 residual=0.00016315079119522125
DEBUG:root:i=227 residual=0.0001611715561011806
DEBUG:root:i=228 residual=0.00015922135207802057
DEBUG:root:i=229 residual=0.00015729668666608632
DEBUG:root:i=230 residual=0.0001554009213577956
DEBUG:root:i=231 residual=0.00015352647460531443
DEBUG:root:i=232 residual=0.00015167582023423165
DEBUG:root:i=233 residual=0.00014985307643655688
DEBUG:root:i=234 residual=0.00014805344108026475
DEBUG:root:i=235 residual=0.00014627669588662684
DEBUG:root:i=236 residual=0.00014452292816713452
DEBUG:root:i=237 residual=0.0001427957322448492
DEBUG:root:i=238 residual=0.00014108861796557903
DEBUG:root:i=239 residual=0.00013940288044977933
DEBUG:root:i=240 residual=0.00013773991668131202
DEBUG:root:i=241 residual=0.00013610217138193548
DEBUG:root:i=242 residual=0.0001344802149105817
DEBUG:root:i=243 residual=0.00013288416084833443
DEBUG:root:i=244 residual=0.00013130543811712414
DEBUG:root:i=245 residual=0.00012974876153748482
DEBUG:root:i=246 residual=0.0001282093726331368
DEBUG:root:i=247 residual=0.00012669437273871154
DEBUG:root:i=248 residual=0.00012519520532805473
DEBUG:root:i=249 residual=0.00012371786579024047
DEBUG:root:i=250 residual=0.00012226005492266268
DEBUG:root:i=251 residual=0.00012081860040780157
DEBUG:root:i=252 residual=0.00011939663818338886
DEBUG:root:i=253 residual=0.00011799144704127684
DEBUG:root:i=254 residual=0.00011660709424177185
DEBUG:root:i=255 residual=0.00011523742432473227
DEBUG:root:i=256 residual=0.00011388548591639847
DEBUG:root:i=257 residual=0.00011255012941546738
DEBUG:root:i=258 residual=0.00011123278818558902
DEBUG:root:i=259 residual=0.00010993158502969891
DEBUG:root:i=260 residual=0.00010864684008993208
DEBUG:root:i=261 residual=0.00010737829143181443
DEBUG:root:i=262 residual=0.00010612665937514976
DEBUG:root:i=263 residual=0.00010488880070624873
DEBUG:root:i=264 residual=0.00010366635251557454
DEBUG:root:i=265 residual=0.0001024613666231744
DEBUG:root:i=266 residual=0.00010126989218406379
DEBUG:root:i=267 residual=0.00010009349352912977
DEBUG:root:i=268 residual=9.893121750792488e-05
DEBUG:root:i=269 residual=9.77837698883377e-05
DEBUG:root:i=270 residual=9.665059769758955e-05
DEBUG:root:i=271 residual=9.553349082125351e-05
DEBUG:root:i=272 residual=9.442601003684103e-05
DEBUG:root:i=273 residual=9.333569323644042e-05
DEBUG:root:i=274 residual=9.225730173056945e-05
DEBUG:root:i=275 residual=9.119306196225807e-05
DEBUG:root:i=276 residual=9.013988164952025e-05
DEBUG:root:i=277 residual=8.910153701435775e-05
DEBUG:root:i=278 residual=8.807294216239825e-05
DEBUG:root:i=279 residual=8.705870277481154e-05
DEBUG:root:i=280 residual=8.605805487604812e-05
DEBUG:root:i=281 residual=8.506963786203414e-05
DEBUG:root:i=282 residual=8.409330621361732e-05
DEBUG:root:i=283 residual=8.312884165206924e-05
DEBUG:root:i=284 residual=8.217326831072569e-05
DEBUG:root:i=285 residual=8.123247971525416e-05
DEBUG:root:i=286 residual=8.030100434552878e-05
DEBUG:root:i=287 residual=7.938157068565488e-05
DEBUG:root:i=288 residual=7.847467350075021e-05
DEBUG:root:i=289 residual=7.757751882309094e-05
DEBUG:root:i=290 residual=7.669001206522807e-05
DEBUG:root:i=291 residual=7.581407407997176e-05
DEBUG:root:i=292 residual=7.495137833757326e-05
DEBUG:root:i=293 residual=7.409582030959427e-05
DEBUG:root:i=294 residual=7.325134356506169e-05
DEBUG:root:i=295 residual=7.24166093277745e-05
DEBUG:root:i=296 residual=7.159150118241087e-05
DEBUG:root:i=297 residual=7.077769259922206e-05
DEBUG:root:i=298 residual=6.997250602580607e-05
DEBUG:root:i=299 residual=6.917809514561668e-05
DEBUG:root:i=300 residual=6.839065463282168e-05
DEBUG:root:i=301 residual=6.761590339010581e-05
DEBUG:root:i=302 residual=6.684752588625997e-05
DEBUG:root:i=303 residual=6.609057891182601e-05
DEBUG:root:i=304 residual=6.534261774504557e-05
DEBUG:root:i=305 residual=6.460091390181333e-05
DEBUG:root:i=306 residual=6.387032044585794e-05
DEBUG:root:i=307 residual=6.314581696642563e-05
DEBUG:root:i=308 residual=6.243080861167982e-05
DEBUG:root:i=309 residual=6.172637222334743e-05
DEBUG:root:i=310 residual=6.102960469434038e-05
DEBUG:root:i=311 residual=6.0341011703712866e-05
DEBUG:root:i=312 residual=5.965817399555817e-05
DEBUG:root:i=313 residual=5.898566450923681e-05
DEBUG:root:i=314 residual=5.832064198330045e-05
DEBUG:root:i=315 residual=5.766164758824743e-05
DEBUG:root:i=316 residual=5.701329791918397e-05
DEBUG:root:i=317 residual=5.636969581246376e-05
DEBUG:root:i=318 residual=5.57365310669411e-05
DEBUG:root:i=319 residual=5.5109205277403817e-05
DEBUG:root:i=320 residual=5.448934462037869e-05
DEBUG:root:i=321 residual=5.387647615862079e-05
DEBUG:root:i=322 residual=5.327027247403748e-05
DEBUG:root:i=323 residual=5.267312371870503e-05
DEBUG:root:i=324 residual=5.2081551984883845e-05
DEBUG:root:i=325 residual=5.149559729034081e-05
DEBUG:root:i=326 residual=5.091820639790967e-05
DEBUG:root:i=327 residual=5.034788773627952e-05
DEBUG:root:i=328 residual=4.9783771828515455e-05
DEBUG:root:i=329 residual=4.9224057875107974e-05
DEBUG:root:i=330 residual=4.8673897254047915e-05
DEBUG:root:i=331 residual=4.812825864064507e-05
DEBUG:root:i=332 residual=4.7590903704985976e-05
DEBUG:root:i=333 residual=4.7057696065166965e-05
DEBUG:root:i=334 residual=4.6531706175301224e-05
DEBUG:root:i=335 residual=4.601059481501579e-05
DEBUG:root:i=336 residual=4.549611185211688e-05
DEBUG:root:i=337 residual=4.498893395066261e-05
DEBUG:root:i=338 residual=4.448606341611594e-05
DEBUG:root:i=339 residual=4.3988875404465944e-05
DEBUG:root:i=340 residual=4.3496740545379e-05
DEBUG:root:i=341 residual=4.301232911529951e-05
DEBUG:root:i=342 residual=4.253320730640553e-05
DEBUG:root:i=343 residual=4.205787263344973e-05
DEBUG:root:i=344 residual=4.158832598477602e-05
DEBUG:root:i=345 residual=4.112701935810037e-05
DEBUG:root:i=346 residual=4.066760448040441e-05
DEBUG:root:i=347 residual=4.0215832996182144e-05
DEBUG:root:i=348 residual=3.976615334977396e-05
DEBUG:root:i=349 residual=3.932518666260876e-05
DEBUG:root:i=350 residual=3.888809806085192e-05
DEBUG:root:i=351 residual=3.8454862078651786e-05
DEBUG:root:i=352 residual=3.8028654671506956e-05
DEBUG:root:i=353 residual=3.7605288525810465e-05
DEBUG:root:i=354 residual=3.718736843438819e-05
DEBUG:root:i=355 residual=3.6774028558284044e-05
DEBUG:root:i=356 residual=3.636586188804358e-05
DEBUG:root:i=357 residual=3.596257374738343e-05
DEBUG:root:i=358 residual=3.556281444616616e-05
DEBUG:root:i=359 residual=3.5167133319191635e-05
DEBUG:root:i=360 residual=3.477800055406988e-05
DEBUG:root:i=361 residual=3.4393964597256854e-05
DEBUG:root:i=362 residual=3.401335561648011e-05
DEBUG:root:i=363 residual=3.363542418810539e-05
DEBUG:root:i=364 residual=3.326293153804727e-05
DEBUG:root:i=365 residual=3.289305095677264e-05
DEBUG:root:i=366 residual=3.2530195312574506e-05
DEBUG:root:i=367 residual=3.21693041769322e-05
DEBUG:root:i=368 residual=3.181384818162769e-05
DEBUG:root:i=369 residual=3.145989830954932e-05
DEBUG:root:i=370 residual=3.1113515433389693e-05
DEBUG:root:i=371 residual=3.0767740099690855e-05
DEBUG:root:i=372 residual=3.042855314561166e-05
DEBUG:root:i=373 residual=3.0090639484114945e-05
DEBUG:root:i=374 residual=2.9758592063444667e-05
DEBUG:root:i=375 residual=2.9430364520521834e-05
DEBUG:root:i=376 residual=2.9104656277922913e-05
DEBUG:root:i=377 residual=2.878522536775563e-05
DEBUG:root:i=378 residual=2.8466878575272858e-05
DEBUG:root:i=379 residual=2.8151993319625035e-05
DEBUG:root:i=380 residual=2.7841560950037092e-05
DEBUG:root:i=381 residual=2.7534357286640443e-05
DEBUG:root:i=382 residual=2.7229545594309457e-05
DEBUG:root:i=383 residual=2.693058195291087e-05
DEBUG:root:i=384 residual=2.6633297238731757e-05
DEBUG:root:i=385 residual=2.6340168915339746e-05
DEBUG:root:i=386 residual=2.6049181542475708e-05
DEBUG:root:i=387 residual=2.5762017685337923e-05
DEBUG:root:i=388 residual=2.547822987253312e-05
DEBUG:root:i=389 residual=2.519718873372767e-05
DEBUG:root:i=390 residual=2.4920940632000566e-05
DEBUG:root:i=391 residual=2.4647866666782647e-05
DEBUG:root:i=392 residual=2.4376702640438452e-05
DEBUG:root:i=393 residual=2.4106526325340383e-05
DEBUG:root:i=394 residual=2.3843133021728136e-05
DEBUG:root:i=395 residual=2.3580081688123755e-05
DEBUG:root:i=396 residual=2.3321745175053366e-05
DEBUG:root:i=397 residual=2.3064038032316603e-05
DEBUG:root:i=398 residual=2.2810618247603998e-05
DEBUG:root:i=399 residual=2.255957224406302e-05
DEBUG:root:i=400 residual=2.2312771761789918e-05
DEBUG:root:i=401 residual=2.2068014004616998e-05
DEBUG:root:i=402 residual=2.1823119823238812e-05
DEBUG:root:i=403 residual=2.158560891984962e-05
DEBUG:root:i=404 residual=2.1346833818824962e-05
DEBUG:root:i=405 residual=2.111371395585593e-05
DEBUG:root:i=406 residual=2.0882003809674643e-05
DEBUG:root:i=407 residual=2.0652318198699504e-05
DEBUG:root:i=408 residual=2.0425834009074606e-05
DEBUG:root:i=409 residual=2.020163810811937e-05
DEBUG:root:i=410 residual=1.9979199350927956e-05
DEBUG:root:i=411 residual=1.976107341761235e-05
DEBUG:root:i=412 residual=1.9543453163350932e-05
DEBUG:root:i=413 residual=1.9328941561980173e-05
DEBUG:root:i=414 residual=1.911869185278192e-05
DEBUG:root:i=415 residual=1.890842577267904e-05
DEBUG:root:i=416 residual=1.869984043878503e-05
DEBUG:root:i=417 residual=1.8495657059247606e-05
DEBUG:root:i=418 residual=1.8293923858436756e-05
DEBUG:root:i=419 residual=1.809269087971188e-05
DEBUG:root:i=420 residual=1.7895437849801965e-05
DEBUG:root:i=421 residual=1.769799564499408e-05
DEBUG:root:i=422 residual=1.7504253264633007e-05
DEBUG:root:i=423 residual=1.7315136574325152e-05
DEBUG:root:i=424 residual=1.7125003068940714e-05
DEBUG:root:i=425 residual=1.6935944586293772e-05
DEBUG:root:i=426 residual=1.6750269423937425e-05
DEBUG:root:i=427 residual=1.656902168178931e-05
DEBUG:root:i=428 residual=1.638621324673295e-05
DEBUG:root:i=429 residual=1.6208157830988057e-05
DEBUG:root:i=430 residual=1.602928932697978e-05
DEBUG:root:i=431 residual=1.585487370903138e-05
DEBUG:root:i=432 residual=1.5680045180488378e-05
DEBUG:root:i=433 residual=1.550904562463984e-05
DEBUG:root:i=434 residual=1.5339712263084948e-05
DEBUG:root:i=435 residual=1.5172819985309616e-05
DEBUG:root:i=436 residual=1.5006519788585138e-05
DEBUG:root:i=437 residual=1.484337462898111e-05
DEBUG:root:i=438 residual=1.4680225831398275e-05
DEBUG:root:i=439 residual=1.451928801543545e-05
DEBUG:root:i=440 residual=1.4362094589159824e-05
DEBUG:root:i=441 residual=1.4203859791450668e-05
DEBUG:root:i=442 residual=1.404900922352681e-05
DEBUG:root:i=443 residual=1.3894718904339243e-05
DEBUG:root:i=444 residual=1.3744362149736844e-05
DEBUG:root:i=445 residual=1.359338239126373e-05
DEBUG:root:i=446 residual=1.344482006970793e-05
DEBUG:root:i=447 residual=1.32992181534064e-05
DEBUG:root:i=448 residual=1.3155035048839636e-05
DEBUG:root:i=449 residual=1.3010933798796032e-05
DEBUG:root:i=450 residual=1.2867833902419079e-05
DEBUG:root:i=451 residual=1.27261564557557e-05
DEBUG:root:i=452 residual=1.258886004507076e-05
DEBUG:root:i=453 residual=1.2452002010832075e-05
DEBUG:root:i=454 residual=1.2314540981606115e-05
DEBUG:root:i=455 residual=1.2181523743493017e-05
DEBUG:root:i=456 residual=1.2046907613694202e-05
DEBUG:root:i=457 residual=1.1916876246687025e-05
DEBUG:root:i=458 residual=1.1785854439949617e-05
DEBUG:root:i=459 residual=1.1657877621473745e-05
DEBUG:root:i=460 residual=1.1530800293257926e-05
DEBUG:root:i=461 residual=1.1403147254895885e-05
DEBUG:root:i=462 residual=1.1279747013759334e-05
DEBUG:root:i=463 residual=1.1157451808685437e-05
DEBUG:root:i=464 residual=1.1036651812901255e-05
DEBUG:root:i=465 residual=1.0915485290752258e-05
DEBUG:root:i=466 residual=1.079707271856023e-05
DEBUG:root:i=467 residual=1.0679666957003064e-05
DEBUG:root:i=468 residual=1.0561921953922138e-05
DEBUG:root:i=469 residual=1.0446934538776986e-05
DEBUG:root:i=470 residual=1.0334350008633919e-05
DEBUG:root:i=471 residual=1.022213382384507e-05
DEBUG:root:i=472 residual=1.011068707157392e-05
DEBUG:root:i=473 residual=9.999514986702707e-06
DEBUG:root:i=474 residual=9.891193258226849e-06
DEBUG:root:i=475 residual=9.783355380932335e-06
DEBUG:root:i=476 residual=9.6770972959348e-06
DEBUG:root:i=477 residual=9.571341252012644e-06
DEBUG:root:i=478 residual=9.466367373534013e-06
DEBUG:root:i=479 residual=9.366161066282075e-06
DEBUG:root:i=480 residual=9.262440471502487e-06
DEBUG:root:i=481 residual=9.16175758902682e-06
DEBUG:root:i=482 residual=9.062349818123039e-06
DEBUG:root:i=483 residual=8.962949323176872e-06
DEBUG:root:i=484 residual=8.865173185768072e-06
DEBUG:root:i=485 residual=8.769921805651393e-06
DEBUG:root:i=486 residual=8.673954653204419e-06
DEBUG:root:i=487 residual=8.578535926062614e-06
DEBUG:root:i=488 residual=8.484613317705225e-06
DEBUG:root:i=489 residual=8.393701136810705e-06
DEBUG:root:i=490 residual=8.302825335704256e-06
DEBUG:root:i=491 residual=8.213876753870863e-06
DEBUG:root:i=492 residual=8.12317830423126e-06
DEBUG:root:i=493 residual=8.034648089960683e-06
DEBUG:root:i=494 residual=7.945961442601401e-06
DEBUG:root:i=495 residual=7.861197445890866e-06
DEBUG:root:i=496 residual=7.780555279168766e-06
DEBUG:root:i=497 residual=7.690730853937566e-06
DEBUG:root:i=498 residual=7.607182396895951e-06
DEBUG:root:i=499 residual=7.524095508415485e-06
DEBUG:root:i=500 residual=7.442438800353557e-06
DEBUG:root:i=501 residual=7.361918051174143e-06
DEBUG:root:i=502 residual=7.282281785592204e-06
DEBUG:root:i=503 residual=7.203785571618937e-06
DEBUG:root:i=504 residual=7.125534011720447e-06
DEBUG:root:i=505 residual=7.047422059258679e-06
DEBUG:root:i=506 residual=6.970416961848969e-06
DEBUG:root:i=507 residual=6.8958302108512726e-06
DEBUG:root:i=508 residual=6.8196891334082466e-06
DEBUG:root:i=509 residual=6.74534840072738e-06
DEBUG:root:i=510 residual=6.671047685813392e-06
DEBUG:root:i=511 residual=6.599600510526216e-06
DEBUG:root:i=512 residual=6.528200174216181e-06
DEBUG:root:i=513 residual=6.456385563069489e-06
DEBUG:root:i=514 residual=6.387566372723086e-06
DEBUG:root:i=515 residual=6.318125997495372e-06
DEBUG:root:i=516 residual=6.250274509511655e-06
DEBUG:root:i=517 residual=6.18161311649601e-06
DEBUG:root:i=518 residual=6.11337009104318e-06
DEBUG:root:i=519 residual=6.049089279258624e-06
DEBUG:root:i=520 residual=5.981376943964278e-06
DEBUG:root:i=521 residual=5.918331680732081e-06
DEBUG:root:i=522 residual=5.853885340911802e-06
DEBUG:root:i=523 residual=5.789278930024011e-06
DEBUG:root:i=524 residual=5.728364612878067e-06
DEBUG:root:i=525 residual=5.666429387929384e-06
DEBUG:root:i=526 residual=5.605057140201097e-06
DEBUG:root:i=527 residual=5.5437794799217954e-06
DEBUG:root:i=528 residual=5.482459982886212e-06
DEBUG:root:i=529 residual=5.422826234280365e-06
DEBUG:root:i=530 residual=5.363915988709778e-06
DEBUG:root:i=531 residual=5.306634193402715e-06
DEBUG:root:i=532 residual=5.248466550256126e-06
DEBUG:root:i=533 residual=5.191532636672491e-06
DEBUG:root:i=534 residual=5.136444997333456e-06
DEBUG:root:i=535 residual=5.079299171484308e-06
DEBUG:root:i=536 residual=5.0240050768479705e-06
DEBUG:root:i=537 residual=4.971335783920949e-06
DEBUG:root:i=538 residual=4.916246780339861e-06
DEBUG:root:i=539 residual=4.862388323090272e-06
DEBUG:root:i=540 residual=4.808914582099533e-06
DEBUG:root:i=541 residual=4.758046543429373e-06
DEBUG:root:i=542 residual=4.7076887312869076e-06
DEBUG:root:i=543 residual=4.657604677049676e-06
DEBUG:root:i=544 residual=4.605378762789769e-06
DEBUG:root:i=545 residual=4.555318355414784e-06
DEBUG:root:i=546 residual=4.505621745920507e-06
DEBUG:root:i=547 residual=4.4563071242009755e-06
DEBUG:root:i=548 residual=4.4082194108341355e-06
DEBUG:root:i=549 residual=4.360421826277161e-06
DEBUG:root:i=550 residual=4.3121435737702996e-06
DEBUG:root:i=551 residual=4.266117230145028e-06
DEBUG:root:i=552 residual=4.21952836404671e-06
DEBUG:root:i=553 residual=4.1749217416509055e-06
DEBUG:root:i=554 residual=4.1317762224934995e-06
DEBUG:root:i=555 residual=4.083786734554451e-06
DEBUG:root:i=556 residual=4.041138254251564e-06
DEBUG:root:i=557 residual=3.9959236346476246e-06
DEBUG:root:i=558 residual=3.9531391848868225e-06
DEBUG:root:i=559 residual=3.9107185330067296e-06
DEBUG:root:i=560 residual=3.868005478580017e-06
DEBUG:root:i=561 residual=3.827399723377312e-06
DEBUG:root:i=562 residual=3.78452841687249e-06
DEBUG:root:i=563 residual=3.7431441342050675e-06
DEBUG:root:i=564 residual=3.7032559703220613e-06
DEBUG:root:i=565 residual=3.66306767318747e-06
DEBUG:root:i=566 residual=3.621921223384561e-06
DEBUG:root:i=567 residual=3.5831208151648752e-06
DEBUG:root:i=568 residual=3.54511576006189e-06
DEBUG:root:i=569 residual=3.5067239423369756e-06
DEBUG:root:i=570 residual=3.4682045679801377e-06
DEBUG:root:i=571 residual=3.4324757507420145e-06
DEBUG:root:i=572 residual=3.394154646230163e-06
DEBUG:root:i=573 residual=3.3567343962204177e-06
DEBUG:root:i=574 residual=3.320176801935304e-06
DEBUG:root:i=575 residual=3.284766762590152e-06
DEBUG:root:i=576 residual=3.2500738598173484e-06
DEBUG:root:i=577 residual=3.214592197764432e-06
DEBUG:root:i=578 residual=3.179665782226948e-06
DEBUG:root:i=579 residual=3.144535639876267e-06
DEBUG:root:i=580 residual=3.117713504252606e-06
DEBUG:root:i=581 residual=3.07836671709083e-06
DEBUG:root:i=582 residual=3.043905053345952e-06
DEBUG:root:i=583 residual=3.0116473226371454e-06
DEBUG:root:i=584 residual=2.9776012979709776e-06
DEBUG:root:i=585 residual=2.947346274595475e-06
DEBUG:root:i=586 residual=2.9151410672056954e-06
DEBUG:root:i=587 residual=2.8823712909797905e-06
DEBUG:root:i=588 residual=2.8528029361041263e-06
DEBUG:root:i=589 residual=2.8209437914483715e-06
DEBUG:root:i=590 residual=2.789380232570693e-06
DEBUG:root:i=591 residual=2.759834160315222e-06
DEBUG:root:i=592 residual=2.728985464273137e-06
DEBUG:root:i=593 residual=2.7024711926060263e-06
DEBUG:root:i=594 residual=2.671321226443979e-06
DEBUG:root:i=595 residual=2.6427012471685885e-06
DEBUG:root:i=596 residual=2.6132233870157506e-06
DEBUG:root:i=597 residual=2.585041329439264e-06
DEBUG:root:i=598 residual=2.557167135819327e-06
DEBUG:root:i=599 residual=2.531201516831061e-06
DEBUG:root:i=600 residual=2.502727511455305e-06
DEBUG:root:i=601 residual=2.475189148753998e-06
DEBUG:root:i=602 residual=2.4495270736224484e-06
DEBUG:root:i=603 residual=2.4227583708125167e-06
DEBUG:root:i=604 residual=2.3943971427797806e-06
DEBUG:root:i=605 residual=2.3696688913332764e-06
DEBUG:root:i=606 residual=2.3428278836945537e-06
DEBUG:root:i=607 residual=2.3202987904369365e-06
DEBUG:root:i=608 residual=2.2923397864360595e-06
DEBUG:root:i=609 residual=2.271258608743665e-06
DEBUG:root:i=610 residual=2.2462716060545063e-06
DEBUG:root:i=611 residual=2.221403292423929e-06
DEBUG:root:i=612 residual=2.202270707130083e-06
DEBUG:root:i=613 residual=2.1718649350077612e-06
DEBUG:root:i=614 residual=2.1509156340471236e-06
DEBUG:root:i=615 residual=2.124424099747557e-06
DEBUG:root:i=616 residual=2.1036962607468013e-06
DEBUG:root:i=617 residual=2.0819090877921553e-06
DEBUG:root:i=618 residual=2.056774519587634e-06
DEBUG:root:i=619 residual=2.0339348338893615e-06
DEBUG:root:i=620 residual=2.0131114979449194e-06
DEBUG:root:i=621 residual=1.9920132672268664e-06
DEBUG:root:i=622 residual=1.969267486856552e-06
DEBUG:root:i=623 residual=1.95015513781982e-06
DEBUG:root:i=624 residual=1.9271544715593336e-06
DEBUG:root:i=625 residual=1.903101065181545e-06
DEBUG:root:i=626 residual=1.8836161643775995e-06
DEBUG:root:i=627 residual=1.8644807369128102e-06
DEBUG:root:i=628 residual=1.8436492155160522e-06
DEBUG:root:i=629 residual=1.8234612753076362e-06
DEBUG:root:i=630 residual=1.8031355466519017e-06
DEBUG:root:i=631 residual=1.785175186341803e-06
DEBUG:root:i=632 residual=1.767990511325479e-06
DEBUG:root:i=633 residual=1.7452779275117791e-06
DEBUG:root:i=634 residual=1.7281031432503369e-06
DEBUG:root:i=635 residual=1.7090205801650882e-06
DEBUG:root:i=636 residual=1.690699832579412e-06
DEBUG:root:i=637 residual=1.6727060483390233e-06
DEBUG:root:i=638 residual=1.6526163335583988e-06
DEBUG:root:i=639 residual=1.6357076901840628e-06
DEBUG:root:i=640 residual=1.617415023247304e-06
DEBUG:root:i=641 residual=1.599909751348605e-06
DEBUG:root:i=642 residual=1.5841668528082664e-06
DEBUG:root:i=643 residual=1.5669220374547876e-06
DEBUG:root:i=644 residual=1.5508538808717276e-06
DEBUG:root:i=645 residual=1.5323154229918146e-06
DEBUG:root:i=646 residual=1.5156407471295097e-06
DEBUG:root:i=647 residual=1.4983982055127854e-06
DEBUG:root:i=648 residual=1.481594722463342e-06
DEBUG:root:i=649 residual=1.4662033436252386e-06
DEBUG:root:i=650 residual=1.453104800930305e-06
DEBUG:root:i=651 residual=1.4355017583511653e-06
DEBUG:root:i=652 residual=1.4202686315911706e-06
DEBUG:root:i=653 residual=1.4077746754992404e-06
DEBUG:root:i=654 residual=1.3906014828535262e-06
DEBUG:root:i=655 residual=1.3772860256722197e-06
DEBUG:root:i=656 residual=1.3602272019852535e-06
DEBUG:root:i=657 residual=1.3443130910673062e-06
DEBUG:root:i=658 residual=1.3323752909855102e-06
DEBUG:root:i=659 residual=1.3151864095561905e-06
DEBUG:root:i=660 residual=1.3019293874094728e-06
DEBUG:root:i=661 residual=1.2872280876763398e-06
DEBUG:root:i=662 residual=1.2731889000860974e-06
DEBUG:root:i=663 residual=1.2586406228365377e-06
DEBUG:root:i=664 residual=1.247690988748218e-06
DEBUG:root:i=665 residual=1.2324320550760604e-06
DEBUG:root:i=666 residual=1.2215374454171979e-06
DEBUG:root:i=667 residual=1.2055879778927192e-06
DEBUG:root:i=668 residual=1.1935990187339485e-06
DEBUG:root:i=669 residual=1.1833579947051476e-06
DEBUG:root:i=670 residual=1.1711828165061888e-06
DEBUG:root:i=671 residual=1.1551720717761782e-06
DEBUG:root:i=672 residual=1.1422683883210993e-06
DEBUG:root:i=673 residual=1.1297029232082423e-06
DEBUG:root:i=674 residual=1.1164993338752538e-06
DEBUG:root:i=675 residual=1.1077530643888167e-06
DEBUG:root:i=676 residual=1.0929325071629137e-06
DEBUG:root:i=677 residual=1.0806334103108384e-06
DEBUG:root:i=678 residual=1.069889890459308e-06
DEBUG:root:i=679 residual=1.0624211199683486e-06
DEBUG:root:i=680 residual=1.0499994687052094e-06
DEBUG:root:i=681 residual=1.03960269370873e-06
DEBUG:root:i=682 residual=1.0239760968033806e-06
DEBUG:root:i=683 residual=1.0128073881787714e-06
DEBUG:root:i=684 residual=1.0159304792978219e-06
DEBUG:root:i=685 residual=9.950388175639091e-07
INFO:root:rank=0 ranking=7.0149e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 ranking=7.0149e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 ranking=1.0552e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 ranking=3.1755e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 ranking=2.2040e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 ranking=1.6027e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 ranking=1.6026e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 ranking=1.6023e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 ranking=1.6020e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 ranking=1.6020e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
   ```

   Task 2, part 1:
   ```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 ranking=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 ranking=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 ranking=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 ranking=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 ranking=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 ranking=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 ranking=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 ranking=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 ranking=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 ranking=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
   ```

   Task 2, part 2:
   ```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 ranking=0.0000e+00 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 ranking=0.0000e+00 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 ranking=0.0000e+00 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 ranking=0.0000e+00 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 ranking=0.0000e+00 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 ranking=0.0000e+00 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 ranking=0.0000e+00 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 ranking=0.0000e+00 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 ranking=0.0000e+00 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 ranking=0.0000e+00 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america

   ```

1. Ensure that all your changes to the `pagerank.py` and `README.md` files are committed to your repo and pushed to github.

1. Get at least 5 stars on your repo.
   (You made trade stars with other students in the class.)

   > **NOTE:**
   > 
   > Recruiters use github profiles to determine who to hire,
   > and pagerank is used to rank user profiles and projects.
   > Links in this graph correspond to who has starred/followed who's repo.
   > By getting more stars on your repo, you'll be increasing your github pagerank, which increases the likelihood that recruiters will hire you.
   > To see an example, [perform a search for `data mining`](https://github.com/search?q=data+mining).
   > Notice that the results are returned "approximately" ranked by the number of stars,
   > but because "some stars count more than others" the results are not exactly ranked by the number of stars.
   > (I asked you not to fork this repo because forks are ranked lower than non-forks.)
   >
   > In some sense, we are doing a "dual problem" to data mining by getting these stars.
   > Recruiters are using data mining to find out who the best people to recruit are,
   > and we are hacking their data mining algorithms by making those algorithms select you instead of someone else.
   >
   > If you're interested in exploring this idea further, here's a python tutorial for extracting GitHub's social graph: <https://www.oreilly.com/library/view/mining-the-social/9781449368180/ch07.html> ; if you're interested in learning more about how recruiters use github profiles, read this Hacker News post: <https://news.ycombinator.com/item?id=19413348>.

1. Submit the url of your repo to sakai.

   Each part is worth 2 points, for 12 points overall.
