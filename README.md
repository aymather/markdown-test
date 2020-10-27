# Virality

## Overview
This dashboard uses a uniq algorithm for determining whether or not a post should be considered "viral".

The first step in this algorithm is determining the posts "virality" metric, which is an intersection between relevance and engagement, and will be explained more in depth later.

The second step is to calculate the sample mean and standard deviation. Once you've done that, you can create your own confidence interval / threshold for what constitutes a "viral post". Currently, the dashboard uses a 95% confidence interval (which is the same as 2 standard deviations above the mean).

It is important to note that this algorithm sits atop an assumption. It assumes that post engagement across time is normally distributed. I will go into more detail later as to what this means, but this is currently a limitation of this algorithm and should eventually be sorted out.

## Calculating Virality

"Virality" is an arbitrary metric we have determined specifically for our purposes. Again, it is a mathmatical intersection between relevance and engagement. In its essence, it can be broken down into these steps:

1. Find the number of days since the post was created.
2. Divide the number of likes by the number of days since the post was created to get an average "likes per day".
3. Starting with a coefficient of 1, multiply each days "likes per day" by the coefficient, but on each subsequent day divide the previous coefficient by 2 - this will mirror an exponential decay of "likes per day" value.
4. Sum the remaining "likes per day" values together to get your virality metric.

#### Example

Say we have a post that was created 3 days ago and currently has 60 likes - I like working with round numbers ;)

```
var days_since_posted = 3;
var likes = 60;
```

##### Get the likes per day
```
var likes_per_day = likes / days_since_posted;
// likes_per_day = 20
```

##### Loop through each day multiplying by exponentially decreasing coefficient
```
var coefficient = 1;
var total = 0;
for(let i = 0; i < days_since_posted; i++) {
    total += likes_per_day * coefficient;
    coefficient /= 2;
}

// total = 35
```

So let's walk through this step by step.

- In the first loop, the coefficient is 1 and the likes_per_day is 20. Because we really want to emphasize recency, we want to give more value to posts that have high engagement, but that are also recent. So on the first day, we're going to give full value to those likes, aka multiply it by our coefficient of 1. So, so far we have a total virality of 20.
- In the second loop, we are now on the second day out. Since these are still relatively recent, we still want to give value to these likes, but not as much value. So we divide our coefficient by 2 (getting 0.5) and multiply that by our likes_per_day, resulting in a value of 10 for the day. Add that to our total and so far we have a virality of 30.
- On our third day, we're starting to get a little far out from our post being "relevant" so we again divide our coefficient by 2 (getting 0.25) and multiply that by our likes_per_day to get a value of 5. Add that to our total and we get our final virality score of 35.

The point of this is to create a scenario in which the longer a post has remained "out there" the more its virality score is taxed. This will put heavy emphasis on high engagement posts that are very recently posted, and very low emphasis on low engagement posts that were posted years ago. It will also place a medium emphasis on exceedingly high engaging posts that were posted a very long time ago - also a medium emphasis on relatively low engaging posts that were posted today. Because relevance and engagement are tethered in this way, in theory, we should mostly see posts that are both highly relevant and highly engaging.

## When is a post considered "Viral"

So now we have a metric to determine a posts "virality", but how much does that actually tell us? We can sort all of the posts by their virality metric, but it would be more meaningful if we had some way of being able to point out, with certainty, which posts should be considered "a viral post". Simply looking at the post with the highest virality score, does not actually mean that that post should be considered "viral". It could very well be the case that we have no posts in our dataset that are going "viral". So how do we determine that?

In order to understand how to calculate this, one must first understand the difference between a population "truth" and a sample "truth".
- A population "truth" is the true metric about the world. In other words, if in theory you could aggregate every facebook post ever, and calculate the mean number of likes across all posts, you would have the "true" mean.
- An estimated "truth" is an approximation of the same metric. In otherwords, because we do not have access to every facebook post ever, we gather a subset and calculate the mean of those posts, and say that it is our best approximation.

The reason we have to understand this is because we're dealing with this problem that, yes we may have a post that is our highest engaging post ever, but does that truly mean that it is "viral"? In otherwords, if we could compare it to the true population mean/standard deviation, would it still hold up?

But again, we run into another problem (statistics has a lot of these) which is that we never have access to the true population mean. So... we have to make an assumption...

Let's say that we have our database of all the posts that we've gathered over the past year, and we also have a list of posts that we just gathered today. In our database, let's say that we have 10,000 posts and in our fresh list we have 50. The assumption we're going to make is that our database is a representation of the "true" population, because it is the best approximation that we have. And we want to figure out if any of our new posts are "viral" or not.

The first thing we're going to do is find the mean and standard deviation of our population(our database)'s virality metric, and we're going to assume that that is our best representation of the "truth".

So how do we know to consider whether or not something is "viral"?

Well, to answer that we have to understand something about distributions...

In any distribution, the obviously sits directly at the center. If you were to encapsulate anything that lies in 1 standard deviation above and below that mean, you would capture 64% of the data in your set. Go 2 standard deviations above and below the mean and you're capturing 95% of your data. 3 Standard deviations and you've got 97.5% of your data. So as you can see, the further you get from the mean, the less likely you are to find one of your datapoint. In other words, it is extremely rare to find a data point that is 99% above your mean. In fact, you have a 1% chance of any data point being above/below your mean, and a 0.5% chance of it being the one that is above (this sort-of illustrates the difference between a one-sided and a two-tailed ttest).

Back to our example...

Since we have our assumed population mean and standard deviation now, we can set our own threshold as to what should be considered "viral". The gold standard for a significant finding in science, is a 95% confidence interval, meaning anything that is 2 standard deviations above the mean. Now, this is NOT the same thing as a ttest, because we are not comparing two sample sets, we are comparing a sample set to a single post - but hey, it's the best we can do.

So let's say that we have a mean virality of 100, and a standard deviation of 10. Then 2 standard deviations above the mean would be a virality of 120. Meaning that a post has a 2.5% chance of having a virality over 120. We get the 2.5% by understanding that 2 standard deviations above and below the mean captures 95% of our data, which would leave 2.5% of our data above that and the other 2.5% below that. And since we're only interested in data that is above our mean, we ignore the 2.5% on the lower-bound.

So now all we have to do, since we have a number that represents 2 standard deviations above our population mean, is find the virality of each of the posts in our fresh new list, and compare that virality to our population. If the virality is greater than that threshold then we consider it "viral", and if it does not exceed our threshold then it is not considered "viral".
