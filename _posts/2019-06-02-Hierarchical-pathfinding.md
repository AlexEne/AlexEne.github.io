---
layout: post
---

Pathfinding gave me a lot of problems as I went with the clasic A* approach for my game. This worked well except one case. When you don't have a path to a certain point, you might end up traversing all the blocks of a world desperately trying to find one. This can be quite time consuming, and even for a relatively small 100x100 world, you might spend `7ms` searching for a path that doesn't exist. `7ms` is a huge amount of time from a game frame of `16ms` so I had to do something about this.

## Intro

There are simple solutions to this problem, one of the most popular ones, that I actually ended up implementing is HPA* or as the original paper calls it, Near-Optimal Hierarchical Pathfinding. But in reality it's just Hirerarchical A*.  
This is best explained in the paper [here](https://webdocs.cs.ualberta.ca/~mmueller/ps/hpastar.pdf).

However, I did some changes in order to adapt that paper to my use case -- a minecraft-like world, with multiple levels. That paper only solves it for the 2D case,  but it's easily extended to a 3D-block world.

## HPA*

So how does it work?  

The main idea of hierarchical A* is to (you guessed it), create a hierarchy first. So we divide our world into bigger cells, and then compute the path and connection points in between these big cells.

![split map](/images/hierarchical_pathfinding/split_world.png "A map being split into high-level cells")

As you can see in the image above, the world is split into 10x10 big cells (the yellow cells). The nice thing about this is that you don't need to tune anything (except maybe the high-level cell sizes). For now let's keep them at 10x10 and we have this world divided into high-level cells.

Once we did this, we proceed and find the connection points in between high level cells. These are the points on the edges. There can be quite a lot of them in the case of an open field and here the paper goes one step further. It makes a sort of doors between two adjacent high-level cells. This way it can only keep 2 connection points for each door over a certain size. I didn't do that but it can be a good optimization.

So right now we have a series of connection points for the high level cells. You can see them with the blue in the picture below. We also added some obstacles (black cells).

![connection points](/images/hierarchical_pathfinding/generated_connection_points.png)

Now we add connection points in between these cells. First we start with the external connections. These external connections are between two adjacent high-level cells. All blue cells have an external connection between themselves and the blue cell they are adjacent to that sits on another high-level yellow cell.
After we've found and cached all external connections, we need to handle the internal connections. These are done by just finding a path with HPA* and checking if all points are in a cell.

In the image below we can see the internal connections for the red cell.
Red and blue connections are the same thing, I used a different color to make things a bit more visible.

![internal connections](/images/hierarchical_pathfinding/internal_connections.png)

## Getting a path

Now that we have our high-level cells and they have connections between them and internal connections we just need to explore two things when we are searching for a path.

1. From the starting point find all possible high-level connections that we can start from
2. Traverse the high-level graph until we reach the high-level cell containing the destination.
3. Once we reached that end high-level cell, do a low-level A* to find if from the entry point we have a path to the destination.

As our entity moves through the world it will encounter a cell that's connected to another far away cell. This is for the case where we need to traverse a high-level cell. In that case we have to call A* pathfinding again for that high-level cell.

It is also possible to cache the path and just querry it as it saves us a A* search for a 10x10 cell. This is what I do and if you have spare memory to cache these paths I highly recommend doing so.

For example a path from the start point `S` to the destination `D` will look like this:
- The orange cells are part of low-level paths.
- The red cells are connected in the high-level pathfinding grid. For them we either have low-level paths cached or we compute them as needed. When we reach the first cell touched by that red arrow.

![generated path](/images/hierarchical_pathfinding/path_found.png)

## Depth & digging

Handling depth is trivial, for each cell (not only the edges) we check if there is a cell below or above that we are connected to. These cells are kept in case we need to descend to a lower level as we search for a path to our destination.  
Handling terrain destruction is simple as we only need to re-compute the high-level connections in the cell where the digging happened. 

## The end

So this is our short journey into HPA*. It is a great way to speed up pathfinding for your games, especially if you're using A* as this comes at an easy integration with your existing pathfinding code. You have to change the core code just slightly and most of the work is done in the initial high-level cell generation.  
Other ways to speed up pathfinding is using alternative data structures and things like nav meshes.  

I also explained this on my stream a while back so if you like this post in a video format you can watch that explanation[here](https://youtu.be/qSbSb8vMbLI?t=915)

