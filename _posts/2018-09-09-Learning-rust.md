---
layout: post
---

While I didn't post anything on my blog I have been working on a lot of fun projects in my spare time. Most notably, I have been learning Rust. Part of doing that resulted in a few neat side-projects that I will talk about here in more detail.
This is mostly my journey on learning a new language, I am sure there are other ways, but this worked for me.  
I started with the rust [book](https://doc.rust-lang.org/book/) and then I continued to learn using a project-based approach, and this worked great for me.  
Below you will see a few of the projects I did along the way with a few words on how they helped me learn various parts of the language.

## Particles

Project: [https://github.com/AlexEne/rust-particles](https://github.com/AlexEne/rust-particles)

After learning the basics, I have started porting one of my old C++ projects to rust. For me this was a great way to compare the two languages in a real-life situation, where I had to use OpenGL, SDL, and other C libraries.  
I don't fully recommned starting with this kind of project since it might not be the smoothest introduction to a new language, but it worked great for me.  

Most of this project is done using compute shaders, so here I mostly learned how to use other C libraries from Rust.

![particles](/images/particles.png)

## Chip8 Emulator

Project: [https://github.com/AlexEne/rust-chip8](https://github.com/AlexEne/rust-chip8).  

I found out that doing a chip8 emulator is one of the best ways to start learning a new language. 
The reason it works so well is because it only uses a few language constructs: for/while/match. It also gets you used to the simple data structures like arrays and maps. You also go in a bit more detail by taking a dependency on another crate (in my case minifb).

It's also a lot of fun since you can find binaries for chip8 games that you can later run inside your emulator.

![rust-chip8](/images/rust-chip8.png)

## Advent of code

[Advent of code](https://adventofcode.com/) is a programming competition running in at the end of the year, and it consists of daily challenges starting from 1 December to 25 December.  
This is great to learn about data structures, file I/O, string manipulation.
I have none of my solutions from last year, but I did finish quite a few of the challenges, but since it was near the holidays I didn't manage to finish it.  
It's really fun when more people are participating and you have a mini-leaderboard where you battle with your friends.

![advent-of-code](/images/advent-of-code.png)


## Raytracing in one weekend

Project: [https://github.com/AlexEne/raytracing-rs](https://github.com/AlexEne/raytracing-rs)

This is a really popular topic nowdays, with new tech like RTX popping up. I stumbled upon the [Raytracing in one weekend](https://www.amazon.co.uk/gp/product/B01B5AODD8) book and this was another fun project that I did in rust, probably the first one where I used parallelism (using [rayon](https://github.com/rayon-rs/rayon)).
You get to learn about multithreading, a bit of ray-sphere intersections, materials and basic raytracing theory.  
I highly recommend the following books: [Raytracing the next week](https://www.amazon.co.uk/Ray-Tracing-Next-Week-Minibooks-ebook/dp/B01CO7PQ8C/) and [Raytracing the rest of your life](https://www.amazon.co.uk/gp/product/B01DN58P8C/).
Both have great information to get you started and then if you really are into this field, you can move on to [Physically Based Rendering from Theory to Implementation](https://www.amazon.co.uk/Physically-Based-Rendering-Theory-Implementation/dp/0123750792)

In this project I learned how threads work in rust and how pleasant and comforting is to get thread-related errors at compile time.

The Raytracing in one weekend book is true to its title, and after one weekend you end up with this kind of image:

![raytracing-rs](/images/raytracing-rs.png)


## Making a game in Rust

The last one in the list for me is doing a game. I've been thinking about this new game idea for a while and I wanted to see how Rust works for games.
This game is currently without a title, but the gameplay is close to gnomoria or similar games.

This is my biggest rust project, at about 6000 lines of code, it includes a super simple _engine_ and ECS architecture. This architecture was mostly inspired by the Overwatch GDC talk that presented how overwatch was built on ECS and how everything worked together.

It uses the following libraries:
- ImGUI
- SDL
- OpenGL
- Serde
- Rayon
- and a few others

ECS worked great for me and now I have a bunch of features at this early stage:
- Terrain generation
- Combat
- Build tree
- Task system
- Hunger/Thirst and other effects.
- Serialization
- Armor and equipment

Working on this I learned about traits, serialization, profiling (more on this below)

They are in various degrees of completion but until now everything works great together.

![dwarves](/images/dwarves_with_helmets_eating_baguettes.gif)

## Rust-hawktracer

Project [https://github.com/AlexEne/rust_hawktracer](https://github.com/AlexEne/rust_hawktracer)

Hawktracer is a lightweight intrusive profiler developed by my coleague at Amazon that was open-sourced a while ago [https://github.com/amzn/hawktracer](https://github.com/amzn/hawktracer).  
While working on my game I had some performance issues that were not easy to diagnose using a sampling profiler and since the original hawktracer project offers a C-API, I have started this crate in order to make rust integration more pleasant.  

It has no runtime overhead if you disable profiling since everything is macro-based and if you disable it you don't get any code generated.  

I learned a lot about macros, binding generation, cmake crate. Currently I am still working on this in order to get it at an acceptable quality to be an actual crate. 

You can find more info on how to integrate it on the above github page, and I encourage you to send feedback.

![rust-hawktracer](/images/rust-hawktracer.png)

## Conclusion

This was my journey in learning rust, taking a project-based approach.
I will continue writing more detailed posts about the things I am working on (especially the game).
