---
layout: post
title: "Cardinality Estimation Benchmark"
---

*Author: [Parimarjan Negi](https://parimarjan.github.io),
[Ryan Marcus](https://rcmarcus.info), Andreas Kipf*

There has been a lot of interest in using Machine Learning models for cardinality estimation. The motivating application, often, is to estimate the sizes of the subplans that a query optimizer encounters when searching for the best execution plan. In the most simplified setting, a better query plan may need to process smaller sized intermediate results, thereby utilizing fewer resources, and executing faster. Several approaches have shown that you can consistently outperform DBMS estimators, often by orders of magnitude in terms of average estimation accuracy. However, improving estimation accuracy may not neccessarily improve an optimizer's final query plan, as highlighted in the following simple example[^estimation_plan_quality].

![Plan Cost Intuition](/assets/ceb/CEB-blog-intuition.png)

We utilize a novel programmatic templating scheme[^templates] to generate over 15K challenging queries on two databases (IMDb, and StackExchange). More crucially, we provide a clean API to evaluate cardinality estimations on all of a query’s subplans with respect to their impact on query plans.

## Example

Consider query 1a66 from CEB-IMDb, shown below. A cardinality estimator provides the size estimate for each of its subplans (everyconnected subgraph of the join graph is a potential subplan we consider). CEB contains true cardinalities for all these subplans.

![CEB query example](/assets/ceb/CEB-blog-eg1.jpeg)

The subplan cardinality estimates can be fed into PostgreSQL, which gives us the best plan for these <i> estimates </i>. The cost of this plan using true cardinalities is the Postgres Plan Cost[^ppc]. Below, we visualize these plans when using the PostgreSQL cardinality estimates (left), which is almost 7x worse than using the true cardinalities (right).

![CEB query plan example](/assets/ceb/1a66-plans.jpeg)

This whole process is automated in the [CEB repo](https://github.com/learnedsystems/ceb) --- you just need to provide the subplan estimates. The colorbar goes from green, i.e., cheap nodes (scan or join operators) to expensive nodes. Thus, looking for the red nodes immediately shows us why the estimates messsed up --- PostgreSQL underestimated cardinalities of two key nodes, and thus, its nested loop join was significantly bad --- since the true cardinalities of these nodes were large, and would therefore require a lot more processing. This is a common pattern of PostgreSQL cardinality underestimates resulting in bad plans --- in fact, most of the challenging Join Order Benchmark cases fall into this category. But, we also see examples in CEB where PostgreSQL gets a worse plan because of overestimates, for instance consider query 2a61 below

![CEB query plan example2](/assets/ceb/2a61-plans.jpeg)

# Why is this benchmark needed?

The Join Order Benchmark (JOB) did a great job of highlighting why TPC- style synthetic benchmarks may not be enough for evaluating query optimization, in particular, the impacts of cardinality estimation. The queries in JOB illustrate the challenges, but there are too few of them. Here is a table comparing the key properties of our benchmark, compared to JOB.


There are several reasons why we would want a larger benchmark.

* <b> Supervised learning / Query driven models. </b> Models such as MSCN, learn based
on a representative workload of queries; Workloads with <= 4 queries per
template, such as JOB, are not great for training such models. By specializing to the known query
patterns, such models can achieve excellent performance without explicitly modeling the data distribution. They can achieve excellent performance at a fraction of the
storage / inference costs of any data driven method (e.g., NeuroCard models for IMDb went
    to several 100MBs, while MSCN models we used were < 2MB). Query driven
models are also easier to adapt to all types of queries, such as those
involving self-joins, ILIKE filters etc. which none of the data driven models
support.

* <b> Deep learning models, and edge cases. </b>
For traditional cardinality estimation models, which were based on analytical formulas, we could be confident of their functioning, including shortcomings, based on intuitive analysis. A lot of the newer models are based on deep learning based black box techniques. Even when a model does well on a particular task --- it does not guarantee that it will have a predictably good performance in another scenario. At the same time, there is a promise of huge performance improvements. One way to gain confidence in these models is to show that they work well across significantly different, and challenging scenarios. Thus, the techniques used for developing CEB aim to create queries with a lot of edge cases, and challenges, which should be useful to study when these models work well, and when they don't. This should let us study their performance in various contexts --- changing workloads, changing data, different training set sizes, changing model parameters, and so on.

<!--But, these models are essentially complex black boxes, and a lot-->
<!--remains to be studied about when we can be confident in their predictions.  But, with deep learning based models, it is hard-->
  <!--to predict when things may go wrong. Therefore, we require, large, diverse-->
  <!--workloads, full of edge cases which may help us to identify thwe weakenesses-->
  <!--of such models, so that we may be able to devise methods to solve them, and-->
  <!--therefore, gain confidence in them.-->

<!--We also provide other commonly used datasets, such as JOB, JOB-M, JOB-light,-->
   <!--and their cardinalities + in the same format. Why is new needed. JOB-light-->
   <!--example; JOB-M and JOB are more challenging, BUT: ; In particular, most of-->
   <!--these queries run quite fast on PostgreSQL + indexes ---- link to our paper;-->
   <!--Another new benchmark --- link ziniu --- was published last week, with similar-->
   <!--motivations; Moreover, they find Postgres Plan Cost as a useful metric to-->
   <!--evaluate the queries as well.-->

<!--Traditional models have well defined assumptions, like column independence. Therefore, their errors are also well understood, and approaches.-->

<!--Redundancy *good*. Provide smaller subset of queries as well. Not done enough-->
<!--analysis to know if we really need these many queries etc., or what is a good-->
<!--subset. Certainly, for initial evaluations, you should not use the whole workload at once.-->

<!--For instance, as we show in X, as query distributions vary, the estimation accuracy of thee models can still be significantly better than DBMS heuristics, but its query performance can get unpredictably worse. Other recent work Y also shows how these models can have drastically varying performance as you update the data, or change the query templates, and so on.-->

<!--In order for such models to get practical acceptance, we need to understand when their estimates are unreliable, and how these impact query plans. To do this, we introduce easily usable tools to evaluate the impact of cardinality estimates on query optimization.-->

<!--A standard way to evaluate cardinality estimation models is Q-Error.-->


<!--Given the cardinality estimates for all of a query’s subplans, we can then ask how good is the query plan that would be generated by a DBMS using these. In Figure X, we show the query plan for one query using the estimates of PostgreSQL; Along with the plan, for every node we also show the true value and the estimate (rounded to the nearest thousand). This gives us an insight into the thinking of the optimizer behind choosing the given plan.-->

<!--In Figure Y, we see what would be the plan chosen by the optimizer given true cardinalities. This lets us better analyze how the cardinality mis-estimates impact the optimizer.-->

<!--These plots are automatically generated with each run.-->

<!--Why do we need more queries?-->

<!--Traditional, histogram based models are well understood, so we can be reasonably sure of their behavior, including their drawbacks, after seeing their performance on a limited set of queries. For instance, JOB served as a great benchmark to highlight the edge cases that can cause significant query performance degradation.-->

<!--But, with increasingly more complex ML models, it is less clear how to analyze them. We generate a lot of queries in a pseudo-random fashion to maximize the chances of generating edge cases that can trip up these models in surprising ways; thus while the average query in the CEB workload may not be very challenging to optimize, it will have several of these edge cases, and we notice this by observing how the 99th percentile runtimes get significantly slower across the board.-->

# Learned Models

We provide a simple featurization scheme for queries, and data loaders for PyTorch, which we use to train several known supervised learning models in the [CEB repo](http://github.com/learnedsystems/CEB). Based on our formulation, and importance, of the Plan Costs, we derived a differentiable loss function that can be a drop in replacement for Q-Error to train neural networks, called Flow-Loss. It is evaluated in detail the paper [here](http://vldb.org/pvldb/vol14/p2019-negi.pdf), and we will discuss it further in a future blog post.

# Next Steps

We would love for you to contribute to further developing CEB, or explore research questions around it. Here are a few ideas:

* <b> Adding more DBs, and query workloads. </b> There are a lot of learned models for cardinality estimation, and very few challenging evaluation scenarios. Thus, it is hard to compare these methods, and to reliably distinguish between the strengths and weakenesses of these approaches. There are two steps to expanding on the evaluation scenarios in CEB. First, we need new databases --- for instance, if you have an interesting real world database in PostgreSQL, then providing a pgdump for it should allow us to easily integrate it into CEB. We provide IMDb, and StackExchange, and plan to add a couple of others in the future. Secondly, once we have a new database, we require query workloads for those. We have provided some tools for automated query generation, but at its core, all such methods would require some representative queries thought through by people familiar with the schema. This is the most challenging step for expanding such benchmarks, and we hope that open sourcing these tools can bring people together to collectively build larger workloads.

* <b> Limits of the learning models. </b> The paper, [Are We Ready For Learned Cardinality Estimation?](http://vldb.org/pvldb/vol14/p1640-wang.pdf) won a Best Paper award in VLDB 2021. They ask several important questions about learned cardinality estimation models, but their experiments are restricted to single table estimations, i.e., without joins. CEB should provide the tools to ask similar questions in the more complex, query optimization use case of these estimators.

<!--* <b> Analyzing the space of optimal plans. </b> How often are different operators used, how many interesting plans are there, space of plans etc. similar to picasso etc.-->

* <b> Data drivel models. </b> Comparing unsupervised learning models (e.g.[Neurocard](https://github.com/neurocard/neurocard), [DeepDB](https://github.com/DataManagementLab/deepdb-public), [Fauce](http://pasalabs.org/papers/2021/VLDB21_Fauce.pdf), [BayesCard](https://arxiv.org/abs/2012.14743)) with the supervised learning models. Some of these approaches are harder to adapt to our full set of queries --- it involves modeling self joins, regex queries, and so on. Working to extend the data driven approaches to these common use cases should be interesting. But we also have also converted other simpler, query workloads, like JOB-M as used in NeuroCard, to our format, and provide scripts to run them and generate the plan costs etc.
* <b> Different execution environments. </b> In the [Flow-Loss paper](http://vldb.org/pvldb/vol14/p2019-negi.pdf), we mainly focused on one particular execution scenario: single threaded execution on NVMe hard drives, which ensured minimum additional noise and variance. We have explored executing these queries in other scenarios, and we notice the runtime latencies fluctuate wildly. For instance, when executing on AWS EB2 storage, even the same query plan latencies can fluctuate due to the I/O bursts. On slower SATA hard disks, we find all the query plans get significantly slower, thus potentially causing many more challenging scenarios for the cardinality estimators. Similar effects are seen when we don’t use indexes. These effects can also be somewhat modeled by different configuration settings --- which would allow Postgres Plan Cost to serve as a viable proxy for latencies in these situations. These queries, and evaluation framework, provide many interesting opportunities to analyze these impacts.

* <b> Alternative approaches to cardinality estimation. </b> Another interesting line of research suggests that query optimizers should not need to rely on precise cardinality estimates when searching for the best plan. This includes [plan bouquets](https://dl.acm.org/doi/10.1145/2588555.2588566), [pessimistic query optimization](https://waltercai.github.io/assets/pessimistic-query-optimization.pdf), [robust query optimization](http://www.vldb.org/pvldb/vol11/p1360-wolf.pdf), and so on. For instance, pessimistic QO approaches have done quite well on JOB. It is interesting to see if they can do equally well on a larger workload, potentially with more edge cases, such as CEB.

* <b> Featurization schemes. </b> Our featurization makes simplifying assumptions, such as already knowing the exact templates that will be used in the queries and so on. This may be reasonable in some practical cases, like templated dashboard queries, but will not support ad-hoc queries. Similarly, models should be able to support self joins without knowing the templates beforehand, and so on.

We envision [CEB](https://github.com/learnedsystems/ceb) as a foundation on which to add additional database backends
besides PostgreSQL, and several other databases and workload of queries, in
order to build a much more challenging set of milestones to build robust and
reliable ML models for cardinality estimation in a query optimizer.
Please contribute!

# Notes

[^estimation_plan_quality]: Intuitively, this is because to get the best plan, you only need the cost of the best plan to be the cheapest. So for instance, large estimation errors on subplans that are not great, would not affect it. There are more such scenarios in the [Flow-Loss paper](http://vldb.org/pvldb/vol14/p2019-negi.pdf).

[^templates]: More details are given [here](https://github.com/learnedsystems/CEB/blob/main/TEMPLATES.md).

[^ppc]: Postgres Plan Cost (PPC) is based on the abstract Plan Cost defined in the excellent ten year old paper, [Preventing Bad Plans by Bounding the Impact of Cardinality Estimation Errors](http://www.vldb.org/pvldb/vol2/vldb09-657.pdf) by Moerkotte et al. They also introduced Q-Error in the paper, which has been commonly used as the evaluation metric of choice in recent cardinality estimation papers. PPC is a useful proxy for query execution latencies in PostgreSQL, based on its cost model, but it is not DBMS specific. For instance, we have the basic ingredients for a hacky MySQL implementation [here](https://github.com/parimarjan/mysql-server). It is useful because executing queries can be very resource intensive, noisy, and so on. Meanwhile, PPC can be computed almost as easily as Q-Error, and it is more closely aligned with the goals of query optimization. And these don't always agree. For instance, we have seen scenarios where an estimator has lower average Q-Error, but higher Postgres Plan Cost. We show its correlation with runtimes, and further discuss the use of the Plan Costs in the [Flow-Loss paper](http://vldb.org/pvldb/vol14/p2019-negi.pdf).
