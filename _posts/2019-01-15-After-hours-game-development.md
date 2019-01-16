---
layout: post
---

In my spare time I am working on a dwarf colony management game that's written in rust.  
I started this project about one year ago and since it has reached this milestone and I didn't abandon it I think it's a good time to look at the curent status.

## What is this about?

![Dwarves](/images/dwarf_game/dwarf_game_january.gif)

So this is how the game looks on the current build.

## General architecture

The game has an **E**ntity-**C**omponent-**S**ystem (**ECS**) architecture. There are many places where this is explained better so I'm going to be lazy and just link them here: 

- Overwatch Gameplay architecture [gdc vault](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)
- RustConf 2018 closing keynote as [video](https://www.youtube.com/watch?v=P9u8x13W7UE) or [text](https://kyren.github.io/2018/09/14/rustconf-talk.html).

My quick TLDR explanation is as follows. This architecture is formed out of three parts:  
- Entities - IDs that refer to different components.  
- Components - Data without behavior.  
- Systems - Behavior that has no state. Each system acts on a collection of components. 

I didn't use [specs](https://github.com/slide-rs/specs) or any of the other rust libraries because I started this project as a learning experiment that slowly transformed into a game. If I would start now I would take a serious look at specs before rolling my own implementation. There are some advantages of existing libraries over what I have most of them around boilerplate code.

To create an entity right now I just edit some mega json file called: `recipe_table.json`. For example the `Baguette` entity recipe looks like this:

```json
    "Baguette": {
      "can_craft": true,
      "items_list": [
        //List of items needed to craft this entity. 
        {
          "entity_type": "Grains",
          "count": 1,
          "processed": false
        }
      ],
      "components": [
        //Data to initialize various components  
        {
          "Item": {
            //Almost all entities have this.
            "can_carry": true,
            "entity_type": "Baguette",
            "blocks_pathfinding": false
          }
        },
        {
          "Renderable": {
            //This is quite clear  
            "texture_type": "Baguette",
            "width": 64,
            "height": 64,
            "layer": 1
          }
        },
        {
          "Consumable": {
            //Consumable entities can be used by dwarves
            //to trigger various effects  
            "modifier": {
              "effect": "Food",
              "effect_value": 60,
              "effect_type": "Instant",
              "ticks_since_last_applied": 0,
              "should_stack": true
            }
          }
        }
      ],
      "workbench_type": "CookingTable"
    }
```

As primitive and ugly it is compared with modern engines UIs, I am really happy with this system. It allows me to quickly create and modify entities. Above all it just works and it allows me to focus on different aspects of the game.

The component properties that you see here aren't deserialized directly into a component, but a `ComponentDescriptor`. A `RenderableComponent` has many other members, but the data in his corresponding `RenderableComponentDescriptor` is enough to instantiate a `RenderableComponent`.

I use [Serde](https://github.com/serde-rs/serde) for any serialization and deserialization job and it is an amazing library that is a joy to use and almost invisible. If by any chance you don't know about it, I recommend you to take a look at some of the example code to get an idea on how it's used.

I am keeping this as open and configurable as possible in the idea that I want to allow people to mod the game. Potentially someone (not me) can even build some fancy UI instead of working directly with this JSON in the future.

There are some limitations to modding and I will keep them moving forward. New components or systems can't be added to the game but the existing components can be combined to create new interesting entities (like a plant that attacks dwarves when they pass near it).

## How much from scratch?

Since people might be interested, here is my `cargo.toml` file:

```toml
[package]
name = "dwarf_game"
version = "0.1.0"
authors = ["Alexandru Ene <alex.ene0x11@gmail.com>"]
edition = "2018"

[dependencies]
sdl2 = "0.31.0"
imgui = "0.0.18"
gl = "0.6.0"
memoffset = "0.1"
png = "0.11.0"
serde = "1.0"
serde_derive = "1.0"
serde_json = "1.0"
derive-new = "0.5"
fnv = "1.0.6"
rayon = "1.0"
rand = "0.5.0"
noise = "0.5.1"
lazy_static = "1.2.0"
log = "0.4"
pretty_env_logger = "0.3"

[dependencies.rust_hawktracer]
version = "0.3.0"
#features=["profiling_enabled"]

[profile.release]
debug = true

[profile.dev]
opt-level=0
```

## Rendering

Rendering is done using OpenGL. I have a small wrapper that provides me with simple abstractions like Texture, ShaderProgram, etc. over [gl-rs](https://github.com/brendanzab/gl-rs). Simplicity is key here and I don't have anything more than's strictly need.

I don't even use texture atlases yet and I just have a bunch of textures dumped into an `assets/images/` folder.

## UI 

Currently the UI is done using [imgui-rs](https://github.com/Gekkio/imgui-rs). Right now it looks too much like some debug UI so I am unsure if this will be used for the actual game UI as well. I am _evaluating_ the follwing options:
- Configuring imgui-rs more (Most likely to be the chosen option)
- Making my own (I really want to avoid this)
- Something else?

The best feature of this UI right now is that I can just click on an entity, select it and view all it's components state so it is an invaluable tool for debugging.

I had to write my own renderer for `gl-rs` for `imgui-rs`. Nothing complicated, I basically just copied the C++ Opengl example from the main imgui project did and translated it to rust.

## Terrain

Probably the most invisible big piece of work right now is the terrain.  
The terrain is actually 3D and generated from a random seed (think minecraft-like).  
Pathfinding works in 3D (so dwarves know when two terrain levels are connected and can go between them), but due to the fact that the current camera is top down and I am such a noob at art, this is impossible to notice unless I explain it.

## Systems

There are a few systems right now at different levels of completion:
- Combat
- Farming
- Health
- Movement
- Pathfinding
- Task assignment
- Task processing
- Needs - hunger and thirst
- and few smaller ones 

Some of them act on a few components, others act on a ton of things. For example the TaskAssignment and TaskProcessing system need to have access to almost all components.

The beauty of this kind of solution is that that complexity and dependency is explicit and contained in one system.
It's clear from just taking a look at the system data that this is complicated, and not hidden away behind other things.

Systems act on collections of components. For example the combat system has the following view:

```rust
#[derive(new)]
pub struct CombatData<'a> {
    dwarf_components: &'a mut DwarfComponentContainer,
    combat_log: &'a mut CombatLog,
    item_components: &'a mut ItemComponentContainer,
    terrain: &'a mut Terrain,
    body_part_components: &'a mut BodyPartComponentContainer,
    armor_components: &'a ArmorComponentContainer,
    transform_components: &'a TransformComponentContainer,
}
```

It also contains the `update_combat` function:
```rust
pub fn update_combat(combat_data: CombatData) -> Vec<Action> {
    ...
}
```

In `world.update()` we just do the following:

```rust
    systems::combat::update_combat(systems::combat::CombatData::new(
        &mut self.dwarf_components,
        &mut self.combat_log,
        &mut self.item_components,
        &mut self.terrain,
        &mut self.body_part_components,
        &self.armor_components,
        &self.transform_components,
    ));
```

This makes it clear what the combat system is modifying (terrain, dwarf components, body parts, etc.).
Armor components are read-only since armor doesn't get damaged in combat yet.

This is way more primitive compared to the way things work in specs or other similar projects where you get an iterator with one dwarf component, one item_component, etc. I have a bit more boilerplate to write in order to get to the same result, but I don't mind that.

## Components

There are a bunch of components such as:
- Renderable
- Armor
- Area
- Farm
- Workbench
- Dwarf
- Consumable
- BodyPart

As I said before all components contain data and no functions (except simple getters/setters).

## Tests

Rust makes so easy to add tests that it's silly not to write them.
I usually tests for bugs I find. Most of them are like integration tests that test how various systems interact. Each time I find a bug I usually try and add a test for it. 

For example, this is one bug I had to solve:  
When dwarves got hungry as they were crafting something, as they went to eat a baguette they left that task in a limbo state and it couldn't be finished. 
 
After I fixed it, it was simple to add a test that checks tasks gets handled properly in that case:  

```rust
#[test]
fn test_hungry_dwarf_eats_and_finishes_task() {
    let mut world = World::new();
    world.spawn_from_entity_type("WorkbenchWoodsmith", Vec3::new(0, 0, 0));
    world.spawn_from_entity_type("WorkbenchAtDestination", Vec3::new(0, 0, 0));

    let dwarf_id = world.spawn_from_entity_type("Dwarf", Vec3::new(0, 1, 0));
    world.spawn_from_entity_type("Baguette", Vec3::new(0, 5, 0));

    world.spawn_from_entity_type("Mattress", Vec3::new(4, 4, 0));
    for _ in 0..5 {
        world.spawn_from_entity_type("Plank", Vec3::new(3, 3, 0));
    }

    let dwarf_component = find_component(world.get_dwaf_components(), &dwarf_id).unwrap();

    //Make the dwarf hungry
    loop {
        world.update(1);
        let dwarf_component = find_component(world.get_dwaf_components(), &dwarf_id).unwrap();
        let stats = dwarf_component.get_stats();
        if stats.hunger <= stats.hunger_limit + 1 {
            break;
        }
    }

    //Create a task for him
    let task = PlaceItemTask::new(world.generate_entity_id(), Vec3::new(6, 6, 0), "Bed");
    world.add_task(Box::new(task));

    for _ in 0..250 {
        world.update(1);
    }

    //Make sure the task ends and the baguette is eaten.
    assert!(world.get_entity_by_type("Baguette").is_none());
    assert!(world.get_entity_by_type("Bed").is_some());
}
```

Almost all tests are instantiating worlds and various entities and scenarios. This means that if a test breaks, I just copy the world initialization code to the main game and I can visualize that test scenario and debug it really easy, pausing the simulaltion, inspecting entities with the debug UI, etc.

## One year

![One year](/images/dwarf_game/first_commit.png)

I started this one year ago and I commited changes to the project constantly.  
How did I manage to stay motivated for this long?

The secret to staying motivated for so long is that there is no secret and I wasn't motivated and hyped to work on it all the time. It's ok to take breaks. I've had periods where I had a lot of activity and pushed a lot of changes followed by weeks I didn't write a single line of code. Last summer I did almost no work on it for almost two months for example.

**When I'm dealing with with big pieces of work that look scary, the only thing that works 100% for me is to just sit down and start writing.**  

I mainly design by experimenting. I also have a bit of advantage here since I've worked in software and gaming for a while so I kind of know how to avoid the most common pitfals. But really, the most important thing is to just sit down and start writing.

Streaming on twitch also helps, even if I don't have a regular schedule. It brings some sort of order and mini-deadlines. It also brings down distractions (even if sometimes there's some chatting going on). I never plan what I stream so I have to sit and work through whatever I said I was going to work on, instead of getting distracted by other things.

Other than that, I have no good answers or advice on this topic.  
It's a first for me too since I usually abandon side projects that take more than 2 months. This one kind of stuck with me.  
Maybe it's also the fact that I really enjoy writing rust?

## What's next?

Things that I want to work on next are:
- Picking a name for it. Really I've been postponing this for way too long.
- Digging and manipulating terrain
- Refine terrain generation and add natural resources
- UI
- More systems (weather)
- AI director for other factions
- Wild animals
- Professions & military
- Temperature
- Water
- Adding more items and recipes
- Many many other things

