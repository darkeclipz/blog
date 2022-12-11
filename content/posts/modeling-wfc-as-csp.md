---
title: "Wave function collapse as CSP, a simple model"
date: 2022-12-10T08:47:11+01:00
draft: true
---

<style>
.small-tile {
    width: 16px;
    height: 16px;
    display: inline;
}
table {
    margin: 0 auto;
}
</style>

The wave function collapse algorithm is a very useful algorithm that is used to generate maps procedurally. 
It is also pretty easy to understand if we look at the problem through the lens of constraint satisfaction problems.
In this post the goal is to define a simple model, which helps to understand how the wave function collapse algorithm works. In a later post we will expand on this model and make it usable for real world scenarios.

## What is the wave function collapse algorithm?

The wave function collapse algorithm is an algorithm that can for example procedurally generate a map based on a set of tiles and rules between those tiles. Of course it is possible to do many other things with the algorith, although I think that explaining the algorithm using the context gives the clearest example.

Since we are talking about tiles, here is a list of a few o them that we are going to use in this model.

| Name       | Tile                                  |
| ---------- | ------------------------------------- |
| Deep water | <img src="/wfc-tiles/deep-water.png"> |
| Water      | <img src="/wfc-tiles/water.png">      |
| Sand       | <img src="/wfc-tiles/sand.png">       |
| Grass      | <img src="/wfc-tiles/grass.png">      |
| Dirt       | <img src="/wfc-tiles/soil.png">       |
| Rock       | <img src="/wfc-tiles/rock.png">       |

