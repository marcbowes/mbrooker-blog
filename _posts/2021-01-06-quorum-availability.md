---
layout: post
title: "Quorum Availability"
---

{{ page.title }}
================

<p class="meta">It's counterintuitive, but is it right?</p>

In our paper [Millions of Tiny Databases](https://www.usenix.org/conference/nsdi20/presentation/brooker), we say this about the availability of quorum systems of various sizes:

> As illustrated in Figure 4, smaller cells offer lower availability in the face of small numbers of uncorrelated node failures, but better availability when the proportion of node failure exceeds 50%. While such high failure rates are rare, they do happen in practice, and a key design concern for Physalia.

And this is what Figure 4 looks like:

![](https://mbrooker-blog-images.s3.amazonaws.com/mtb_fig_4.png)

The context here is that a *cell* is a Paxos cluster, and the system needs a majority quorum for the cluster to be able to process requests. A cluster of one box needs one box available, five boxes need three available and so on. The surprising thing here is the claim that having smaller clusters is actually *better* if the probability of any given machine failing is very high. The paper doesn't explain it well, and I've gotten a few questions about it. This post attempts to do better.

Let's start by thinking about what happens for a cluster of one machine (*n=1*), in a datacenter of *N* machines (for very large *N*). We then fail each machine independently with probability *p*. What is the probability that our one machine failed? That's trivial: it's *p*. Now, let's take all *N* machines and put them into a cluster of *n=N*. What's the probability that a majority of the cluster is available? For large *N*, it's 1 for *p < 0.5*, and 0 for *p > 0.5*. If less than half the machines fail, less than half have failed. If more than half the machines fail, more than half have failed. Ok?

![](https://mbrooker-blog-images.s3.amazonaws.com/quorum_avail_a.png)

Notice how a cluster size of 1 is worse than N up until *p = 0.5* then better after.

What about real practical cluster sizes like *n=3*, 5, and 7? There are three ways we can do this math. In [The Availability of Quorum Systems](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.38.5629&rep=rep1&type=pdf), Peleg and Wool derive closed-form solutions to this problem, along with a whole lot of other interesting facts about quorums, majority quorums and other things. Our second approach is to observe that the failures of the nodes are Bernoulli trials with probability *p*, and therefore we can read the answer to "what is the probability that 0 or 1 of 3 fail for probability *p*" from the distribution function of the [binomial distribution](https://en.wikipedia.org/wiki/Binomial_distribution). Finally, we can be lazy and do it with Monte Carlo. That's normally my favorite method, because it's easier to include correlation and various "what if?" questions as we go.

Whichever way you calculate it, what do you expect it to look like? For small *n* you may expect it to be closer in shape to *n=1*, and for large *n* you may expect it to approach the shape of *n=N*. If that's what you expect, you'd be right.

![](https://mbrooker-blog-images.s3.amazonaws.com/quorum_avail_b.png)

I'll admit that I find this result deeply deeply counter-intuitive. I think it's right, because I've approached it multiple ways, but it still kind of bends my mind a little. That may just be me. I've discussed it with friends and colleagues over the years, and they seem to think it matches their intuition. It's counter-intuitive to me because it suggests that smaller *n* (smaller clusters, or smaller cells in Physalia's parlance) is better for high *p*! If you think a lot of your boxes are going to fail, you may get better availability (not durability, though) from smaller clusters.

Weird.

**Correlation to the rescue!**

It's not often that my statistical intuition is saved by introducing correlation, but in this case it helps. I'd argue that, in practice, that you only lose machines in an uncorrelated Bernoulli trial way for small *p*. Above a certain *p*, it's likely that the failures have some shared cause (power, network, clumsy people, etc) and so the failures are likely to be correlated in some way. In which case, we're back into the game we're playing with Physalia of avoiding those correlated failures by optimizing placement.

In many other kinds of systems, like ones you deploy across multiple datacenters (we'd call that *regional* in AWS, deployed across multiple *availability zones*), you end up treating the datacenters as units of failure. In that case, for 3 datacenters you'd pick something like *n=9* because you can keep quorum after the failure of an entire datacenter (3 machines) and any one other machine. As soon as there's correlation, the math above is mostly useless and the correlation's cause is all that really matters.

Availability also isn't the only thing to think about with cluster size for quorum systems. Durability, latency, cost, operations, and contention on leader election also come into play. Those are topics for another post (or section 2.2 of [Millions of Tiny Databases](https://www.usenix.org/conference/nsdi20/presentation/brooker)).