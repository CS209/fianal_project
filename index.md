---
---

# Sports NBA Win Probability Analysis and Player Performance Metrics

By: Bolei Deng and Matheus Fernandes

## Contents
{:.no_toc}
*  
{: toc}

## Problem Statement 

We established a model to predict the win probablity (WP) of a certain game at certain point based on the play-by-play data and team stats. Our prediction is then compared with true WP of the games with an error with in $\pm5$%. Based on the WP prediction model, we generated the WP curve with respect to time remaining for each game. Based on that, we evaluate players in the league by their ability to add WP to their teams. More specifically, we fit a linear regression model the on court players data with WP prediction. Then the coefficient of this model is extracted as an indication of players contribution to their teams WP factor.


## Introduction and Description of Data
The reasons this project is important is because it provides useful information for betting, gambling, and people who closely follow all of the NBA games.

It is callenging because there are two many factors to consider. How good the team is in offense? how good the team is in defensive? how well the players shoot? Is the style of one team typically suppress the style of another team? Furthermore, the situation becomes more complex during the process of a game. A weaker team with good conditions (at home, good moral, etc) could beat the better team. Then we need to consider how much time remains? How active the team is (rebounds)? How nervous the plays are (turnovers and number of timeouts)? How hot is the team (shooting rate for this game)? As you can see there are enormous amount of factors that will influence the trend of the game. 

 [See the EDA page here for more information.](http://cs209.fer.me/Sports-EDA.html)

The motivation for for this project is to be able to reproduce and improve the prediction probability provided by ESPN. This was defined throught the preliminary EDA by looking at which features where useful in separating the winning team from the loosing teams. As you can see from our EDA, there are some distinct features that provide very useful information in predciting the winning team. For instance, we can see from one of the plotls that the score differce over time provide a lot of information. Futhermore, The defensive rebound difference over the offencive reound difference also provides useful information in predicting which team will win. However, infromation such as offensive rating difference over deffensive rating differnece seems to have little information on which team will win. Therefore we were able to take this information and engineer features that are important for the 

## Literature Review/Related Work

[1] Sammer K. Deshpande and Shane T. Jensen, Estimating an NBA player's impact on his team's chances of winning, J. Quant. ANal. Sports 12(2), 2016 [[PDF HERE](http://cs209.fer.me/PDF/1.pdf)]

    This literature gives us a hint that we can use the win probability changes to evaluate the players in the league. They used Bayesian to model the players impacts on WP and evaluate player based on their contributes. The win probability model they used is very naive, with only two features: time remaining and leading score. They didn't check the performance of their WP model, that's not their main point.


[2] Dennis Lock and Dan Nettleton, Using random forest to estimate win probability before each play of an NFL game, JQAS 10(2), 2014 [[PDF HERE](http://cs209.fer.me/PDF/2.pdf)]

    Win probability is predicted here using random forest model on NFL game. Again the number of feature they used is not too large, around 10. And all the features are related to NFL. 

[3] ESPN: http://www.espn.com/nba/

    We obtained team stats from ESPN's website. We compared are win probabiltiy results with theirs.

[4] Basketball reference: https://www.basketball-reference.com

    We scraped play-by-play data from 2012-13 season to 2017-18 season from this website.

## Modeling Approach and Project Trajectory
The baseline qualitative comparison for our model is a similar metric found in the ESPN website. However, a quantitative baseline comparison for our model is to accurately predict the outcome of the game for the largest duration of the game through the win probability. To go beyond achieving this baseline, we implement a model which via feature engineering and educated model development takes into account information optimal for accurate prediction of win probability. We achieve this by performing extensive EDA analysis and choosing features useful for win separation. As you will see in the results section, we are able to successfully beat this baseline model by maintaining the correct winning probabilty side over the more of the game time than ESPN does.

Train & test splitting issue: when we train and test our model, we notice that, the rows within a single game are highly correlated. So if we randomly separate the data into training and test set, part of a game can be in training set while another part in test part. Then the model will be trained on each game not each row, which gives us wrong model. To avoid this issue and accurately be able to forecast a given game, we must accurately forecast our performance using test-train split. To do an accurate test-train split we must not just randomly select events, but instead we
must select an entire game instance and segregate game instances into test-train sets. This will allow us to test a game it has never seen before and not use the model to learn the game of which the event came from to predict the outcome.

[See Data Scraping and Model Training for more information.](http://cs209.fer.me/Sports-DataGenerationAndModelTraining.html)

## Results, Conclusions, and FutureWork
As mentioned previously, our project is subdivided into two parts: 1. predicting the win probability of a game as it proceeds, and 2. analyzing how valuable a player is based on how the win probability changes when that player is inside the game. The following are the answers to this question for the two parts, respectively:

1. For predicting the win probability of a game as a function of time, we have compared our performance to the ESPN prediction and as can be seen from our results, our prediction is less sensitive to the score and more accurately portrays the winning team over the scope of time than ESPN does. A metric we observe here is what percent of the time is the win probability on the correct side of the 50% line based on the final result. Although when we comapre our prediction to that of the one found on the ESPN website we see that similar behaviour through time, the behaviour of our model seems to be less sensitive and stay more time on the correct side of the prediction line than the one ESPN predicts. Furthermore, this trend even continues through a game that includes overtime, which ESPN's model does extremely poorly on. However, on shortcoming of our model as compared to the ESPN's model is that our model picks a more extreme winning probability at the initial time than ESPN's. This is a result in that we weigh our initial prediction based on the team higher than that of ESPN's. To mititgate that we must consider balancing more the importance of the teams and give more weight in probability prediction to the events happening in the game. 
[See Win Probability Evaluation for more information.](http://cs209.fer.me/Sports-GameWinProbabilityEvaluation.html)

2. For the part predicting how valuable a player is, our results produces a list that is consistent to our expectations of what the most valuable players are. As you can see from our model results for this part, we predict some of the key players such as LeBron James, James Harden, Giannis Antetokounmpo, and Joel Embiid to be top contributors to increasing the WP of their teams when on court. This as you can imagine ic consistent to what popular belief is as they perform very well compared to their team-mates. However, a potential short-coming of our model is that this method only separate a good player from the rest when the rest of the team they are in lags compared to them. For instance, there are many good players in the Golden Sate Warriers that are not shown in this list, but are great none-the-less. This is a result of the fact that the team is in balance and thus will not show their good team-members as when they are in the court it does not add much to the WP. To mititgate this issue, we believe we need to futher incorporate the peformance of the entire team to this metric. In other words, we need to nomalize the current player performance metric by the overall team performance. [See Player Performance Evaluation for more information.](http://cs209.fer.me/Sports-PlayerPerformanceEvaluation.html)



