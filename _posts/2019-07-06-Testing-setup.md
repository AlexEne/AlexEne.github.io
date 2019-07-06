---
layout: post
---

Usually videogame developers have three possible approaches to testing:  
- Hire armies of people to do it for you.
- Early access.
- Hope all is fine.

We all know how good the support in rust is for writing tests and I would like to show you my improved testing setup for the game I'm working on.  

# Testing

I just test the high-level behavior as things evolve quite fast in my game and having super detailed unit tests for all individual parts is not really worth the effort. At the end of the day all I care about is that if I tell a dwarf to dig a whole at a certain position, he digs a whole at that position.  
Time is also a key part of this strategy. I just don't have enough time to have a lot of detailed unit tests so these high-level tests have to do.

In my previous [post](2019-01-15-After-hours-game-development.md) I said:  
_"Almost all tests are instantiating worlds and various entities and scenarios. This means that if a test breaks, I just copy the world initialization code to the main game and I can visualize that test scenario and debug it really easy, pausing the simulation, inspecting entities with the debug UI, etc."_ - me

Copy-pasting things got annoying after some big refactors where I had to check why I was failing a bunch of test cases.  
That prompted me to change my testing setup so here's my __improved__ approach.  

Today my tests look like this:

```rust
#[test]
fn dwarf_eventually_dies_of_hunger() {
    let mut world = World::new();
    world.spawn_from_entity_type("Dwarf", Vec3::new(0, 0, 0));
    
    let mut game = Game::new(world, false);
    game.update(Some(3000));
    
    //RIP
    let world = game.get_world();
    assert!(world.get_entity_by_type("Dwarf").is_none());
}
```

At a first glance this looks exactly as the old tests that I presented in the other blog post.  
However, there's an important difference, it is using the new `Game` struct.  
I can enable the full experience: graphics, window, events, UI just by changing `Game::new(world, false)` to `Game::new(world, true)`.  
This means that I can click on objects, inspect, even change tests as they happen by issuing build orders.  

And here's the above test running:  

![testing](/images/testing/test_hunger.gif)

I even tried hard to have the graphics always on for all tests but, due to a [limitation][imgui_bug] of IMGUI, I can't spawn more than one instance of it on multiple threads so this is out of the question.  

I consider this new setup a big step forward for my productivity in debugging tests and in adding writing new tests as right now it's super easy to check that a test checks the right thing.

[imgui_bug]: (https://github.com/ocornut/imgui/issues/586)
