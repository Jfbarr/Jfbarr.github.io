---
layout: post
title: "Dijkstra Maps"
author: "Jasper Barr"
categories: article
tags: code
image: 2d_game.png
---

Recently I participated in a hackerthon run by CoderOne, in which the aim was to build an autonomous agent to play a custom Bomberman-like game. The aim of the game was rather simple: place bombs to destroy blocks for points, and avoid getting hit. In essence, there were two problems to solve: pathfinding, and deciding when to place bombs. To solve the problem of deciding where to move on the map, I took inspiration from an [article](http://www.roguebasin.com/index.php?title=The_Incredible_Power_of_Dijkstra_Maps) written by the developer of [Brogue](https://en.wikipedia.org/wiki/Brogue_(video_game)), Brian Walker. Brogue utilises _Dijkstra maps_ for enemy pathfinding, otherwise known as _potential flow_ models elsewhere. In the following, I will provide an overview of how to construct and use Dijkstra maps, an overview of my bots logic, and concluding thoughts from the competition. 


# What are Dijkstra maps?

A Dijkstra map is essentially a heightmap that our agent will navigate by moving to the adjacent tile with the lowest height. To create a Dijkstra map, we use the following algorithm:

1. Set each grid cell to a default value of 100
2. Given a set of goals, set the value of each goal to 0
3. For each cell `c` in the grid:
    - If there is an adjacent cell `c_a` with a difference greater than 1, set `c` equal to `c_a + 1`
    - Otherwise, do nothing
4. Repeat steps 1-3 until no changes are recorded

The output will be a grid with 0s at our goal locations, and increasing values as we move away from goals. As it happens, the value of a cell is the $$ L^1 $$ distance to the nearest goal. For example, a 5x5 grid with a goal at the center would return:

```
43234
32123
21012
32123
43234
```

The same grid with goals at each corner would look like:

```
01210
12321
23432
12321
01210
```

To navigate a Dijkstra map, our agent decides to move to the lowest adjacent value. If the lowest value is the current position, then the agent will remain in the same location. If there are multiple options, the agent randomly chooses between them. Note that the default value of 0 can be replaced with any other chosen value. Negative goal values will have the agent seek out goals from further away.

A key observation from the second exmaple is that an agent placed at the centre can navigate to any of the four goals. This demonstrates that the agent already has some capacity to decide between different goals (even if our decision process is random at this point). 

## Combining maps
Given two Dijkstra maps `D_1` and `D_2`, we can define the sum elemntwise by `D_3[i][j] = D_1[i][j] + D_2[i][j]`. Taking our previous examples, this yields

```
43234   01210   44444
32123   12321   44444 
21012 + 23432 = 44444 
32123   12321   44444 
43234   01210   44444
```

The resulting uniform map is not particularly useful, as the agent will simply wander the field randomly. To remedy this, we define multiplication of a Dijkstra map `D` by a scalar `a` by `aD[i][j] = D[i][j] * a`. Multiplying a map by a constant can be thought of multiplying the height of the hills in the Dijkstra map, so that the agent will be more likely to navigate towards the goals in maps with higher coefficients. 

# Agent
## Overview

Our agent relies on the following logic to decide it's action for each turn.

1. If the previous action hasn't been completed, skip its turn (this accounts for desync conditions)
2. Update a list of bombs in play and the tiles that will be hit with an explosion
2. Update an internal list of 'desires' based on game state
3. Generate a list of Dijkstra maps based on game state
4. Create a final 'goal' Dijkstra map a linear combination of the maps from step 3, with coefficients calculated in step 2
5. Check which surrounding tile the agent can move to based on its current position
    * If the action returned is to do nothing, the agent checks if it should place a bomb
    * If the action returned would move the agent into the opponent, the agent does nothing

## Used maps
The maps that we generate are the following:

1. Bomb Dijkstra - Used to make our agent flee from bombs
2. Reward Dijkstra - This allows our agent to know where it should place bombs
3. Safe bomb Dijkstra - Rather than fleeing, this map gives our agent an aversion to tiles within the explosion radius of a bomb
4. High timer bomb Dijkstra - This is the same map as the Safe bomb map, except we exclude bombs that won't explode in the near future
5. Ammo Dijkstra - This allows our agent to seek treasure and ammo on the map
6. Flee Dijkstra - This allows our agent to flee from the opposing player in later stages of the game
7. Centre Dijkstra - This map keeps our agent near the centre of the map in later stages of the game

# Conclusions

Although our agent performed well during test matches, we didn't make it into the quarter finals. The two main reasons for this would be the lack of performance on competition hardware, and the lack of functionality around hunting other players (and avoiding being hunted ourselves). The first point was an unfortunate result of the hardware used to develop our agent being a good deal more powerful than the cloud resources used to run the competition. As a result, our agent suffers a fairly large delay of 3 turns to make a single move. The second issue could have been avoided with better foresight, and a larger sample size of test matches with opposing teams to observe the main strategies that were in play.
