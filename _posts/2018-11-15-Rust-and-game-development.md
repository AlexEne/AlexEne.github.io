# Rust and game development

## Why ?

Rust is excellent for performance crucial applications that run on multi-processor architectures and these two aspects are also critical for game development. Rust has already seen a bunch of interest from games developers like [Chucklefish](https://www.rust-lang.org/pdfs/Rust-Chucklefish-Whitepaper.pdf), [Embark Studios](https://twitter.com/repi/status/1060469377500274689), [Ready at Dawn](https://twitter.com/andreapessino/status/1021532074153394176?lang=en), etc. - but in order to really excel I'd love to organize some structured efforts to improve the ecosystem and I think it would be great if the 2019 roadmap will include game development.

## But what is game development anyway?

Games are made of complex systems where a lot of things usually need to happen in a short amount of time. In game development you have to do your work fast. In games you your work 16 milliseconds fast. If you're lucky you get about 32 milliseconds.

Even if we ignore rendering, you need to do: physics, animation, updating various gameplay systems, AI, pathfinding and it usually doesn't stop here. That's a lot of things that need to happen and that's why C++ is usually language of choice for game engine development.

## The problem space

I will break down the problem into two. After doing this step, we have two problems, but trust me they are a bit easier.

### 1. Big engines

These are represented by the big companies that build their own, equally big engine (Ubisoft, DICE, Epic, Unity, Lumberyard, etc.).

There are a few restrictions here, most of them are written in C++ under the hood (even Unity). Most didn't care about ABI compatibility since it's all compiled at once so this makes communication to a new Rust module less than ideal. I am not saying we need to solve C++ to Rust bindings, but we need to consider how do we solve fitting Rust systems in an existing engine.

Portability to closed systems. This applies to both categories, but it is really important for this one. While Rust is available on many platforms and architectures, that doesn't mean it just works on console X or console Y and it's supported out of the box if you write a `hello world` program.
Console game development is a strange, NDA-filled space, unknown to many, and while things are  moving in the right direction, there are still problems that need to be solved.

These mammoth engines also care about performance. Sure, you might say consoles are powerful today, but as we said in the intro, you only get 16ms and players expect a lot of things to happen in today's AAA game worlds.
To achieve this kind of performance a lot of optimization work is put into data structure layouts, SIMD, custom memory allocators, etc. Some of these are already doing quite well in Rust, but others not so well. For example custom memory allocators for the standard containers has a great RFC, but it's not done yet.

### 2. Small engines / games

Here we have the small-medium sized companies that don't use an off-the-shelf engine, like Chucklefish, Killhouse, and many others (even lone-wolf gamedevs like me, I do a game in my spare time in Rust).

I've put _engines / games_ in the title place since we have cases where the engine is a custom-built thing for one game. Yes it still happens even today and that's fine.

I believe that Rust as a language is ready and has enough maturity and features for this to be possible. As I said in the intro, it's not only me as a mad, game developer who thinks this, others have jumped on board way before me and announced that they will develop their next game fully or by using rust as much as possible.

__Why did mention this category if I think we're there already for most cases?__

Because there are unsolved issues and nuisances here too. One problem that I've found is that you kind of have to be an expert to make a game and if you start with the wrong path you get into somewhat frustrating situations. Rust punishes you for being wrong more than other languages, and it does at compile time so you have to do things right in order for them to work.

There's no _it works by the power of luck_ here and sometimes that feels bad.

People have explained it way better than me here and there are resources available, but it's something to keep in mind - for more info and solutions see Catherine West's excellent [Rustconf 2018 keynote](https://www.youtube.com/watch?v=aKLntZcp27M)

In this space, there are also some Rust game engines but compared to Unity tutorials they have a higher barrier of entry. For example, for the [Amethist engine](https://www.amethyst.rs/), a simple game of Pong starts out with the following code:
```
impl<'a, 'b> SimpleState<'a, 'b> for Pong {
}
```
_What is that?_ You might cry, but have no fear, I kind of had the same reaction when I saw that a state needed two lifetime annotations. `'a` and `'b`. There are good reasons for having them, but for someone who wants to write pong it's a bit scary. It's scary for me too and I've worked in game development for more than 8 years and I write Rust for more than an year quite intensively.

Do I need that even for pong? I would bet that you can rewrite something like Doorkickers or Stardew Valley or any other 2D game in rust without having to annotate many lifetimes.

Amethist is shaping to be a nice engine but if all you want to do is a 2D game with simple rules, you could get away with simpler abstractions.

Possibly my point is that there is enough space for more engines to appear and address various targets.

## The solution

Now that we've seen a bit of the space Rust game development, let's look at what I think would be a solution.

**I propose starting a Game Development focused Working Group.**

So we made the working group, now what?

## What should this Rust Game Development Working Group do?

Besides the usual WG tasks, the role of this working group is to find and tackle systemic problems that game developers face as they write their games in Rust.

Gathering these pain points sorting and distributing them them to the teams that handle different parts of the ecosystem is one role.

Communicating and teaching through tutorials, a status of what the problems encountered are and general info of what's happening in this space.

This isn't going to touch a single area. It impacts multiple parts of the ecosystem and we need to identify and collaborate in solving any pain points found.

Now for the practical steps I will split my solution into two parts. -- _this is an ongoing theme with me splitting things in two parts_

## The short term

For the short term I'd see a focus on the second category. We are almost there, but there are things still missing or confusing.

We need more resources focused on problems game developers face daily. Some of these I'm sure have been solved a few times already.
For example, if you decided to serialize things with Serde, what's the best way of serializing / deserializing a `Vec<Box<SomeTrait>>` object? I've personally spent probably 4-5 evenings on this problem. I'm certainly not the brightest tool in the shed, but it would be nice to have a bit more posts and information shared on how you could solve certain things that people usually hit in programming with Rust in this space.

Tooling is another subject. RLS is really good but unfortunately it competes with years of effort put into IDEs like Visual Studio.

I know that Windows is a platform that usually doesn't get much love, but these days it's the usual development platform used for games. For example, the rust compiler doesn't even compile on windows with debug enabled due to linking problems (tries to link too many objects).

C++ / C# patterns don't directly translate to rust. You can switch from C++ to C# really easy since both accept the same kind of patterns with ease. Rust doesn't like a lot of these (I'm not going to say bad, but let's say risky) patterns. Unclear hierarchies of things, shared mutable ownership, etc. These is a space where the rust core team focuses on anyway and provides great solutions and [advice](http://smallcultfollowing.com/babysteps/blog/2018/09/24/office-hours-1-cyclic-services/), but it's worth mentioning them as a potential problem that game developers will face.

Custom allocator support are a must and almost everyone mentions them so I'd advocate that that needs to have a higher priority and that's why I include it in this section.

## The Long term

This is a bit more painful to get to, not because of technical reasons but because the world is complex. 
Changing things in big organizations or systems is hard but not impossible.

I hope for a future when you can just do `cargo build` and get a binary that runs on a game console of your choice. I think it's a better future for everyone: players will experience less crashes from common avoidable causes and developers are enabled by a modern language.  
Much of the hard work has been done and I don't know of any language features that are needed in order to make this rust game development initiative a success. If there are, we should start discussing them and drafting a RFC.

As a general note, I don't think what's in the long term category has to start after we finish the short term category, but I just feel that it may take longer to move and change incredibly big systems with a lot of moving parts.

## Things that we should think about

It has been suggested to have a sort of consensus around Amethist or another game engine or libraries like specs as the go-to engine/frameworks for game development in Rust.

They are amazing projects and there are advantages in having a one-engine / library focused ecosystem (from the point of discoverability and community), but I think that diversity is important, not only at your workplace and life, but also in the tooling and framework space. Not all games have the same requirements and not all games need engines, so it's important to be open because it's quite challenging to create a one-size fits all solution.

That's not to say that we shouldn't promote these as options and acknowledge progress, but we should discuss if the focus of this working group should be more towards enabling such frameworks and engines to exist rather than having an agreement on a library/engine.

## Conclusion

I am really excited for the core team to announce more structured processes for spinning up working groups in 2019 so that we can move this group forward!

Game development is a field that's already full of many unknowns and risks. The ultimate goal of this Game Development Working Group is to take away as many risks as possible by making Rust for game development a viable and I would hope default option.
