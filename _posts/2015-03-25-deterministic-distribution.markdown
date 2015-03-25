---
layout: post
title:  "Deterministic Distribution"
title-pic: /images/Technology3.jpg
date:   2015-03-25 15:00:00
categories: forecasting distribution algorithm
author: <a href="https://github.com/egoon">Bennie Lundmark</a>
author-pic: //gravatar.com/avatar/c88d8bdfffc14f1a407edd8af12bf14a
---

<style type="text/css">
  p { text-align: justify; text-justify: inter-word; }
</style>

The distribution algorithm is one of the most central in the Videoplaza ad server. In this article I will explain how the current and new algorithms work, why we need to change, and what the up- and down-sides are with deterministic distribution.


### Introduction

Let me start this by explaining a little bit about how the ad server at Videoplaza works.

<img src="/images/deterministic-distribution/ad-serving.png" title="The pink dude is probably you" alt="Ad serving diagram" width="80%">

A Karbon User, usually a sales person at a cable network or something similar, has sold some ad space to an advertiser. She uploads the ad in the Karbon UI, and adds a bunch of rules according to the deal with the advertiser. The rules usually contain a date span, what type of program the ad is shown with, and how many times the ad should be viewed (called Impressions). The ad is then saved to an entity store, where other parts of the ad server can access it.

At some point a TV viewer watches a show that is supported by ads. At every point where there is supposed to be an ad (usually at the start of the show, mid-way and at the end) the ad player makes a request to the distributor for a list of ads to show. The distributor figures out which ads to show (this is called a Selection, more on this later). As the ad player plays the ads, it also records every ad that is shown, every Impression, in the Delivery Metrics store. These impressions are then used to calculate which ads should be shown next.

### Distribution basics (a bit simplified)

In the Distributor there is a list of ads that may be shown. The ads contain a bunch of rules, like start and end date, delivery goal, tags and much more. And there is also a weight for each ad.

The rules are used to filter the ad list for each request. And then the weight is used to determine which of the filtered ads should actually be returned.

The weights are updated periodically, using an algorithm that considers how long the ad is going to be active, how much of it’s delivery goal has already been delivered, how often it is filtered out and more. Weight are usually between 0 and 1, where 0 means that the ad should never be selected (it has probably already reached it’s delivery goal), and 1 means that it should be selected in every request that meets the filter criteria.

The Selector then randomly selects an ad from the list. If the sum of all ads’ weights is less than 1, there is a chance that no ad will be selected. See example 1:

*Example 1:*

<img src="/images/deterministic-distribution/example1.png" alt="Ad1 weight 0.6, Ad2 weight 0.3" width="80%">

*Ad 1 has 60% of being selected, Ad 2 has 30% chance. And there is a 10% chance that no ad will be selected.*

In the case where sum of the ads’ weights is greater than 1, all weights will be reduced proportionally. See example 2:

*Example 2:*

<img src="/images/deterministic-distribution/example2.png" alt="Ad1 weight 0.6, Ad2 weight 0.3, Ad3: weight 0.3 -> Ad1 weight 0.5, Ad2 weight 0.25, Ad3 weight 0.25" width="80%">

*The sum of all weights is 1.2 and has to be scaled down by 5/6. Ad 1 has 50% chance of being selected, and Ad 2 and 3 have 25% chance each.*


### The problem with random forecasting

The current random algorithm, explained above, works very well in most cases. If only a few requests are made, the result is very random, and impossible to forsee. It does however even out in the long run. Especially since the weights are updated periodically based on previous delivery.

In the forecast team we are currently developing a forecasting engine that runs the distributor, using historical data and current account setup, to simulate the behaviour of campaigns in the future. It works pretty well, and gives pretty consistent result between simulations. 

Or, it did, until we developed the feature called Daily Breakdown.

Daily Breakdown lets you not only see the end result of your simulated campaign, but also how much was delivered each day during the forecast. It turned out that the outcome for any specific day could vary as much as 50% between simulations. This makes Daily Breakdown very unreliable, and pretty much useless. When forecasting with Daily Breakdown we want to see an even distribution using fewer requests: 1/100 to 1/1000, and for a shorter period of time: one day rather than the ad’s lifetime.

We tried increasing the sample size, but the reduction in performance was unacceptable. And the result was not as good as we would hope. So we decided to try a more deterministic approach to ad selection. An approach that would produce an even distribution with a lot fewer requests.

## Deterministic Selection

### Concepts

+ *Actual Weight:* The weight given to an ad. The same weight as described in ‘Distribution Basics’ above.
+ *Current Weight:* The weight the selection is based on. This is a function of Actual Weight, the number of times the ad was part of a selection, and the number of times it was selected. The Current Weight is 0 initially for every ad.
+ *Air:* Air represents the chance of nothing being selected. If the sum of all ads’ Actual Weight is less than 1, the air weight is 1 - the sum. 
 
### Algorithm

After the list of ads has been filtered, and if necessary normalised as described above, instead of randomly picking an ad, we use the following procedure:

1. Add each ad’s (and the air’s) Actual Weight to its Current Weight. (a)
2. Select the ad with the highest Current Weight.
3. Reduce the selected ad's Current Weight by 1 (b)
4. If there are several positions in the selection, adjust the selected ad's Current Weight as if it was present in the selection of the other positions as well (c)

(a): An ad that is part of multiple selections without being selected, will have it's Current Weight increased multiple times, and thereby increasing the chance of being selected in future selections.

(b): An ad that is selected will have a smaller chance of being selected again. If the weight was not reduced, the same ad would always be selected.

(c): A selected ad's Current Weight should be the same, regardless of which position it was selected for. Because an ad can not take two positions in the same selection, it will not be part of selection for later positions if it is selected early. It's Current Weight would not be increased as many times as if it was selected last. This must be compensated for.
 
Each iteration, the same amount of weight is added and subtracted from the Current Weights, which makes it a zero-sum algorithm. That is good for when new ads becomes available as time passes. If the Current Weights would increase only, a new ad would not get selected for a very long time as its weight catches up with other ads. If the Current Weights would decrease over time, then 'old' ads would cease to deliver for a long time as a newer ads became available. It also prevents any overflow problems nicely.

Most goals will have a Current Weight of between -1 and 1 most of the time. In certain circumstances it might go above or below that slightly for a short time.

### The downside

This algorithm introduces more state to the distributor: the Current Weights. The Videoplaza Distributor is a highly distributed service, running on multiple servers across several datacenters on (at least) 3 continents. Keeping the state consistent on all those servers is theoretically possible, but will increase latency and reduce throughput significantly. This is why we are using this algorithm in the forecasting enginge only, where the state can be localized for a single forecast to a single server.

### Other considerations

*Multiple Positions:* Quite frequently there are multiple ads in each ad selection. Usually no ad will appear twice in the same selection. After an ad is selected, it is removed from the list (unless it’s air), but the Actual Weights are not changed. Then another selection is made on the remainder of the list, including increasing the Current Weights. To make sure that it does not matter if an ad is selected first or last, the selected (and removed) ad’s Current Weight is also increased.

*Different targeting:* Some ads with wider targeting will be condending with different ads in different selections. This algorithm will still work in these circumstances. See example 2 below.

<style type="text/css">
    td.up{ color: green; }
    td.down{ color: red; }
    tr.filtered{ color: lightgray; }
</style>

### Example 1

<table class="table">
  <tr>
    <th>Ad</th><th>Actual W</th><th>Current W</th><th>Selections</th>
  </tr>
  <tr>
    <td>Ad1</td><td>0.5</td><td>0</td><td>0</td>
  </tr>
  <tr>
    <td>Ad2</td><td>0.25</td><td>0</td><td>0</td>
  </tr>
  <tr>
    <td>Air</td><td>0.25</td><td>0</td><td>0</td>
  </tr>
</table>
<p/>

Calculations are written as Previous Current Weight + Actual Weight [ - 1 if selected]

*Selection 1*

Increase the ads’s Current Weights by their Actual Weight (0 -> 0.5, 0 -> 0.25)

Select heaviest ad (Ad1)

Reduce its weight by 1 (0,5 -> -0.5)

<table class="table">
  <tr>
    <th>Ad</th><th>Actual W</th><th>Calculation</th><th>Current W</th><th>Selections</th>
  </tr><tr>
    <td>Ad1</td><td>0.5</td><td>0 + 0.5 -1</td><td class="down">-0.5</td><td class="up">1</td>
  </tr><tr>
    <td>Ad2</td><td>0.25</td><td>0 + 0.25</td><td class="up">0.25</td><td>0</td>
  </tr><tr>
    <td>Air</td><td>0.25</td><td>0 + 0.25</td><td class="up">0.25</td><td>0</td>
  </tr>
</table>
<p/>

*Selection 2*

Increase Ad1 and Ad2 Current Weights by their Actual Weight (-0.5 -> 0, 0.25 -> 0.5)

Select heaviest ad (Ad2)

Reduce its weight by 1 (0,5 -> -0.5)

<table class="table">
  <tr>
    <th>Ad</th><th>Actual W</th><th>Calculation</th><th>Current W</th><th>Selections</th>
  </tr><tr>
    <td>Ad1</td><td>0.5</td><td>-0.5 + 0.5</td><td class="up">0</td><td>1</td>
  </tr><tr>
    <td>Ad2</td><td>0.25</td><td>0.25 + 0.25 -1</td><td class="down">-0.5</td><td class="up">1</td>
  </tr><tr>
    <td>Air</td><td>0.25</td><td>0.25 + 0.25</td><td class="up">0.5</td><td>0</td>
  </tr>
</table>
<p/>

*Selection 3*

Increase Ad1 and Ad2 Current Weights by their Actual Weight (0 -> 0.5, -0.5 -> -0.25)

Select heaviest ad (Air)

Reduce its weight by 1 (0,75 -> -0.25)

<table class="table">
  <tr>
    <th>Ad</th><th>Actual W</th><th>Calculation</th><th>Current W</th><th>Selections</th>
  </tr><tr>
    <td>Ad1</td><td>0.5</td><td>0 + 0.5</td><td class="up">0.5</td><td>1</td>
  </tr><tr>
    <td>Ad2</td><td>0.25</td><td>-0.5 + 0.25</td><td class="up">-0.25</td><td>1</td>
  </tr><tr>
    <td>Air</td><td>0.25</td><td>0.5 + 0.25 -1</td><td class="down">-0.25</td><td class="up">1</td>
  </tr>
</table>
<p/>

*Selection 4*

Increase Ad1 and Ad2 Current Weights by their Actual Weight (0.5 -> 1, -0.25 -> 0)

Select heaviest ad (Ad1)

<table class="table">
  <tr>
    <th>Ad</th><th>Actual W</th><th>Calculation</th><th>Current W</th><th>Selections</th>
  </tr><tr>
    <td>Ad1</td><td>0.5</td><td>0.5 + 0.5 - 1</td><td class="down">0</td><td class="up">2</td>
  </tr><tr>
    <td>Ad2</td><td>0.25</td><td>-0.25 + 0.25</td><td class="up">0</td><td>1</td>
  </tr><tr>
    <td>Air</td><td>0.25</td><td>-0.25 + 0.25</td><td class="up">0</td><td>1</td>
  </tr>
</table>
<p/>

*Selection 5+*

Iterate from selection 1 and keep increasing number of selections. After every four selections Ad1 will have been selected exactly 50% of the time, and Ad2 exactly 25% of the time. In a predictable pattern.

### Example 2

We have three ads, with different targeting. Ad1 is targeted site-wide, which means it matches all requests.

Reqests alternate between News and Sports. It may seem like Ad1 will push too much on Ad2 because they will compete in the first request, but it will even out fairly quickly.

<table class="table">
  <tr>
    <th>Ad</th><th>Target</th><th>Actual W</th><th>Current W</th><th>Selections</th>
  </tr><tr>
    <td>Ad1</td><td>All</td><td>0.5</td><td>0</td><td>0</td>
  </tr><tr>
    <td>Ad2</td><td>News</td><td>0.5</td><td>0</td><td>0</td>
  </tr><tr>
    <td>Ad3</td><td>Sports</td><td>0.5</td><td>0</td><td>0</td>
  </tr>
</table>
<p/>

*Selection 1* (News) 

Increase Ad1 and Ad2 Current Weights by their Actual Weight

Select heaviest ad (Ad1)

Reduce its weight by 1 (0,5 -> -0.5)

<table class="table">
  <tr>
    <th>Ad</th><th>Target</th><th>Actual W</th><th>Calculation</th><th>Current W</th><th>Selections</th>
  </tr><tr>
    <td>Ad1</td><td>All</td><td>0.5</td><td>0 + 0.5 - 1</td><td class="down">-0.5</td><td class="up">1</td>
  </tr><tr>
    <td>Ad2</td><td>News</td><td>0.5</td><td>0 + 0.5</td><td class="up">0.5</td><td>0</td>
  </tr><tr class="filtered">
    <td>Ad3</td><td>Sports</td><td>0.5</td><td></td><td>0</td><td>0</td>
  </tr>
</table>
<p/>

*Selection 2* (Sports) 

Increase Ad1 and Ad3 Current Weights by their Actual Weight

Select heaviest ad (Ad3)

Reduce its weight by 1 (0,5 -> -0.5)

<table class="table">
  <tr>
    <th>Ad</th><th>Target</th><th>Actual W</th><th>Calculation</th><th>Current W</th><th>Selections</th>
  </tr><tr>
    <td>Ad1</td><td>All</td><td>0.5</td><td>-0.5 + 0.5</td><td class="up">0</td><td>1</td>
  </tr><tr class="filtered">
    <td>Ad2</td><td>News</td><td>0.5</td><td></td><td>0.5</td><td>0</td>
  </tr><tr>
    <td>Ad3</td><td>Sports</td><td>0.5</td><td>0 + 0.5 - 1</td><td class="down">-0.5</td><td class="up">1</td>
  </tr>
</table>
<p/>

*Selection 3* (News) 

Increase Ad1 and Ad2 Current Weights by their Actual Weight

Select heaviest ad (Ad2)

Reduce its weight by 1 (1 -> 0)

<table class="table">
  <tr>
    <th>Ad</th><th>Target</th><th>Actual W</th><th>Calculation</th><th>Current W</th><th>Selections</th>
  </tr><tr>
    <td>Ad1</td><td>All</td><td>0.5</td><td>0 + 0.5</td><td class="up">0.5</td><td>1</td>
  </tr><tr>
    <td>Ad2</td><td>News</td><td>0.5</td><td>0.5 + 0.5 - 1</td><td class="down">0</td><td class="up">1</td>
  </tr><tr class="filtered">
    <td>Ad3</td><td>Sports</td><td>0.5</td><td></td><td>-0.5</td><td>1</td>
  </tr>
</table>
<p/>

*Selection 4* (Sports) 

Increase Ad1 and Ad3 Current Weights by their Actual Weight

Select heaviest ad (Ad1)

Reduce its weight by 1 (1 -> 0)

<table class="table">
  <tr>
    <th>Ad</th><th>Target</th><th>Actual W</th><th>Calculation</th><th>Current W</th><th>Selections</th>
  </tr><tr>
    <td>Ad1</td><td>All</td><td>0.5</td><td>0.5 + 0.5 - 1</td><td class="down">0</td><td class="up">2</td>
  </tr><tr class="filtered">
    <td>Ad2</td><td>News</td><td>0.5</td><td></td><td>0</td><td>1</td>
  </tr><tr>
    <td>Ad3</td><td>Sports</td><td>0.5</td><td>-0.5 + 0.5</td><td class="up">0</td><td>1</td>
  </tr>
</table>
<p/>

Already after four selections, all current weights are back to 0, where we can start iterating again. Ad1 gets selected twice as often as the others even though they have the same weight. But it still get's selected in 50% of all selections it is a part of. 

For this reason the weight calculations must take targeting into consideration (actually we count how often an ad is filtered out and increase the weight accordingly).