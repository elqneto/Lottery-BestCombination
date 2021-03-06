# Application to a real lottery: EuroMillions

![Let's pick the best numbers](http://i.imgur.com/bIOUoRB.png)

Euromillions consist of picking 5 numbers among 50 + 2 stars among 10 stars.

**We'll focus on the 5 numbers** in order to keep this analysis comparable to [Using ML To Pick Your Lottery Numbers](http://nbviewer.ipython.org/url/www.onewinner.me/en/devoxxML.ipynb).

### 1. Data Import

After each draw, the EuroMillions organizers communicate publicly the results:

*  N1, N2, N3 , N4, N5 : 5 numbers of the draw *(in an ascending order)*
*  Number of tickets on each winning rank:
  + Rank1:   5 matches & 2  good stars
  + Rank2:   5 matches & 1  good star
  + Rank3:   5 matches & 0  good star
  + Rank4:   4 matches & 2  good stars
  + Rank5:   4 matches & 1  good star
  + Rank6:   4 matches & 0  good star
  + Rank7:   3 matches & 2  good stars
  + Rank9:   3 matches & 1  good star
  + Rank10:  3 matches & 0  good star
  + Rank8:   2   matches & 2  good stars
  + Rank12:  2   matches & 1  good star
  + Rank11:  1  match & 2  good stars
* Number of Tickets: total number of tickets played *(no matter the number of matches and/or good stars)*

**We will study the results of the last 300 past draws** *(which happened after the changes of May 2011)*
```r
# Data import
setwd('C:/Users/Stéphane/Documents/Euromillions')
Draws = read.table("euromillions_results_300_draws.csv", header=TRUE,sep = ";")
#  Draws = Draws[1:100,] /!\ If less than 8GB of memory, keep just 100 draws.
```

### 2. Data Preparation

Since we decided to focus on the numbers and not on the stars, we need to aggregate all the ranks associated to the same level of correct numbers *(matches)*.

**Problem:** the numbers of tickets per winning rank are not sufficient to compute the number of players with 2,1 or 0 match. 

For instance we can't compute the total number of tickets with 2 matches without the number of tickets with 2 matches & 0 star *(unknown because no prize associated to this result)*.

Hopefully we can estimate these numbers by using the [Winning Probability](http://en.wikipedia.org/wiki/EuroMillions#Prize_structure):

* 2   matches & 0 good star : can be estimated as twice the number of players in the Rank12 
* 1   match & 1 good star : can be estimated as 18 times the number of players in the Rank11
* 1   match & 0 good star : : can be estimated as 36 times the number of players in the Rank11 
* 0   match (no matter the number of good star) : can be estimated as the total number of players - the number of player with at least 1 match.
  
*A better estimation mechanism could be used  ([See Next-Steps](https://github.com/StephaneFeniar/Lottery-BestCombination/blob/master/README.md#next-steps))*

```r
Grid= c(1:50)
Combinations = t(combn(Grid,5)) # 2 118 760 combinations based on the 5 numbers from 1 to 50.

# Compute for each draw, the number of tickets for each number of matches
Draws$NB_TICKETS_5_MATCHES = Draws$RANK1 + Draws$RANK2 + Draws$RANK3
Draws$NB_TICKETS_4_MATCHES = Draws$RANK4 + Draws$RANK5 + Draws$RANK6
Draws$NB_TICKETS_3_MATCHES = Draws$RANK7 + Draws$RANK9 + Draws$RANK10
Draws$NB_TICKETS_2_MATCHES = Draws$RANK8 + Draws$RANK12 + 2 * Draws$RANK12
Draws$NB_TICKETS_1_MATCH = Draws$RANK11 + 18 * Draws$RANK11 + 36 * Draws$RANK11
Draws$NB_TICKETS_0_MATCH = Draws$NB_TICKETS - Draws$NB_TICKETS_5_MATCHES - Draws$NB_TICKETS_4_MATCHES - 
                           Draws$NB_TICKETS_3_MATCHES-Draws$NB_TICKETS_2_MATCHES-Draws$NB_TICKETS_1_MATCH
                           
# Replace the numbers by frequencies in order to have comparable draws.
Draws$FREQ_TICKETS_5_MATCHES = Draws$NB_TICKETS_5_MATCHES / Draws$NB_TICKETS
Draws$FREQ_TICKETS_4_MATCHES = Draws$NB_TICKETS_4_MATCHES / Draws$NB_TICKETS
Draws$FREQ_TICKETS_3_MATCHES = Draws$NB_TICKETS_3_MATCHES / Draws$NB_TICKETS
Draws$FREQ_TICKETS_2_MATCHES = Draws$NB_TICKETS_2_MATCHES / Draws$NB_TICKETS
Draws$FREQ_TICKETS_1_MATCH = Draws$NB_TICKETS_1_MATCH / Draws$NB_TICKETS
Draws$FREQ_TICKETS_0_MATCH = Draws$NB_TICKETS_0_MATCH / Draws$NB_TICKETS
```
#####################################################

### 3. Compute the number of matches between each Draw and each Combination

Before estimating the playing frequency of each combination, we need to compute the number of match between each draw and each combination.

Instead of directly compare each vectors of DrawsNumbers with each row of Combination,s we're gonna convert them to sparse vectors and take the matrix product.

```r
# Compute the number of matches between each Draw and each Combination.
DrawsNumbers = as.matrix( Draws[,c('N1', 'N2', 'N3', 'N4', 'N5')] )
NbDraws = nrow(DrawsNumbers)
NbCombinations = nrow(Combinations)

DrawsNumbersSparse = matrix(0, nrow=NbDraws, ncol=50)
for(i in 1:NbDraws) {DrawsNumbersSparse[i,DrawsNumbers[i,]] = 1}


CombinationsSparse = matrix(0, nrow=NbCombinations, ncol=50)
for(i in 1:NbCombinations) {CombinationsSparse[i,Combinations [i,]] = 1}

Matches = CombinationsSparse %*% t(DrawsNumbersSparse)
rm(CombinationsSparse)
```

### 4. Estimations of the playing frequency of each combination

Now, we know for each draw:

* the frequency of tickets with 0, 1, 2, 3, 4, 5 matches
* the number of matches with all the 2 118 760 possible combinations

For estimating the playing frequency of each combination we need to split the frequency of tickets with **m** matches to all the combinations having **m** matches with the draw.

No matter the draw, the number of combination with *m* matched  = $\binom{45}{5-m} \times \binom{m}{5}$


Then we know for each draw there is:

* 1 combination having 5 matches 
* 225 combinations having 4  matches 
* 9 900 combinations having 3 matches 
* 141 900 combinations having 2 matches 
* 749 975 combinations having 1 match 
* 1 221 759 combinations having 0 match

So for each draw we will estimate the:

*  Playing frequency of each combination with 0 match = Frequency of tickets with 0 match / 1221759 
*  Playing frequency of each combination with 1 match = Frequency of tickets with 1 match / 749975
*  Playing frequency of each combination with 2 matches = Frequency of tickets with 2 matches / 141900
*  Playing frequency of each combination with 3 matches = Frequency of tickets with 3 matches / 9900
*  Playing frequency of each combination with 4 matches = Frequency of tickets with 4 matches / 225
*  Playing frequency of each combination with 5 matches = Frequency of tickets with 5 matches / 1

```r
#Divide the frequency of tickets with m matches by the number combinations having m matches
Draws$FREQ_COMB_0 = Draws$FREQ_TICKETS_0_MATCH / 1221759 
Draws$FREQ_COMB_1 = Draws$FREQ_TICKETS_1_MATCH / 744975 
Draws$FREQ_COMB_2 = Draws$FREQ_TICKETS_2_MATCHES / 141900
Draws$FREQ_COMB_3 = Draws$FREQ_TICKETS_3_MATCHES / 9900
Draws$FREQ_COMB_4 = Draws$FREQ_TICKETS_4_MATCHES / 225
Draws$FREQ_COMB_5 = Draws$FREQ_TICKETS_5_MATCHES / 1


DrawsCombFrequency  = as.matrix( Draws[,c('FREQ_COMB_0',
                                          'FREQ_COMB_1', 
                                          'FREQ_COMB_2', 
                                          'FREQ_COMB_3', 
                                          'FREQ_COMB_4',
                                          'FREQ_COMB_5')] )

ComputePlayingFrequency = function(DrawsCombFrequency, Matches){ 
  PlayingFrequency = matrix(nrow=nrow(Matches), ncol=ncol(Matches)) 
  for(t in 1:ncol(Matches)) { PlayingFrequency[,t] = DrawsCombFrequency[t,Matches[,t]+1] } 
  return(PlayingFrequency)
}

PlayingFrequency = ComputePlayingFrequency (DrawsCombFrequency, Matches)
```

### 5. Aggregation of the estimations


Each of the 300 draws gives an estimated playing frequency for each combination.
Let's aggregate these 300 estimators into a stronger one like a RandomForest aggregates trees.

There are a several way to aggregate these 300 estimators *([See Next-Steps](https://github.com/StephaneFeniar/Lottery-BestCombination/blob/master/README.md#next-steps))*  but here we will do it by a simple mean.

```
# Aggregation of the estimations
PlayingFrequencyAggregated = rowMeans(PlayingFrequency)
```

### 6. Results analysis


Let's take a look at the most played and less played combinations.
```
# Most frequently played combination (Don't play it :o )
Combinations[order(-PlayingFrequencyAggregated)[1],]
[1]  7 13  19 22 24

# Less frequently played combination (Play this one instead :) )
Combinations[order(PlayingFrequencyAggregated)[1],]
[1]  14 30 41 45 49
```

So playing [14 20 41 45 49] instead of [ 7 13 19 22 24] **reduce the risk** of sharing the winning **by 10% !**

We can notice the most played combination [ 7 13 19 22 24] includes:

* only numbers <30 which could be associated to dates.
* numbers which are close on the grid.
* the [number 13](http://en.wikipedia.org/wiki/13_%28number%29#Lucky_and_unlucky).

# Conclusion & Next Steps

**This simple analysis** identifies the less played combination *[14 20 41 45 49]* and **reduces the risk of sharing the winning up to 10%!**

We could go further by :

*  **Improving the way to aggregate the estimations.** 
<br/>Example: if a combination has 5 matches with a draw, we should only keep the estimation associated to this draw.


*  **Performing the analysis in a recursive way:**
  1. Do the analysis like we did and recover an estimated  playing frequency for each combination.
  2. Re-do the analysis by using this distribution instead of an equal distribution for improving the way to "split the frequency of tickets with m matches through all the combinations having m matches" 


*  **Improving the approach by optimizing the global expected winnings.**
<br/> Rather than just reduce the risk of sharing the big prize, we could reduce the risk of sharing all prizes by searching a combination which is rarely played and doesn't shared a lots of numbers with most played combinations.



