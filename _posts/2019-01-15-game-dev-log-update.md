---
layout: post
---

I am working for a while on a dwarf colony management game in my spare time written in rust. This project is moving forward at various speeds since almost exactly 1 year ago and I think that it's time to do a bit of writing on how things are progressing.

## What is this about?

![Dwarves](/images/dwarf_game/dwarf_game_january.gif)

There are many things going on in the gif above so I will write a bit about all the systems that are in place now. Some of them are really early stage, others are reasonably at a stage where I think not much will change.

## General architecture

The game is done more or less from scratch and it has an **E**ntity-**C**omponent-**S**ystem (**ECS**) architecture. There are many places where this is explained better and in more depth, but here is my quick take on ECS:

Entities - IDs that refer to different components.  
Components - Data without behaviour.  
Systems - Behaviour that has no state.  

Each system acts on a collection of components. I didn't really follow the _normal_ ECS architecture as it is described in the excelent Overwatch gameplay architecture talk available on the [gdc vault](https://www.gdcvault.com/play/1024001/-Overwatch-Gameplay-Architecture-and)

Another good presentation about ECS is the talk presented in the RustConf 2018 closing keynote as [video](https://www.youtube.com/watch?v=P9u8x13W7UE) or [text](https://kyren.github.io/2018/09/14/rustconf-talk.html).

I also didn't use [specs](https://github.com/slide-rs/specs) because I started this as a learning experiment that slowly transformed into a game. If I would start now, I would probably take a serious look at specs before rolling my own implementation. There are some advantages mostly around boilerplate code.

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
          "ItemComponent": {
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
            //to trigger various modifiers  
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

As primitive and ugly as it is compared with modern engines UIs, I am really happy with this system. It allows me to quickly add, modify and inspect various entities, and it just works, allowing me to focus on different aspects.

Since we are on the loading data part, I must mention that [Serde](https://github.com/serde-rs/serde) is an amazing library that I'm using to load this data and also serialize the world state (for save games). It is a joy to use and almost invisible.

I am keeping this as open and configurable as possible in the idea that I want to allow people to mod the game. Potentially someone (probably not me) can even build some fancy UI instead of working directly with this JSON in the future. So basically new components or systems can't be added to the game but the existing components could be combined to create new interesting entities (like a flower that generates heat)

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

Rendering is done using OpenGL, I require 3.2 at the moment but there's no reason to not work with even earlier versions.

I have a small wrapper that provides me with simple abstractions like Texture, ShaderProgram, etc. over [gl-rs](https://github.com/brendanzab/gl-rs). Simplicity is key here and I don't have anything more than I strictly need here.

I don't yet use texture atlases, and I just have a bunch of textures dumped into an `assets/images/` folder.

## UI 

Currently the UI is done using [imgui-rs](https://github.com/Gekkio/imgui-rs). Right now it looks too much like some debug UI so I am unsure if this will be used for the actual game UI as well. I am evaluating right now other options:
- Configuring imgui-rs more
- Making my own
- Something else?

One note about the UI is that I had to write my own renderer for gl-rs for imgui. Nothing too fancy, I basically just copied whatever the C++ examples in the main imgui project did and translated it to rust.


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
Here the armor components and transform components are read-only since armor doesn't get damaged in combat yet.

This is way more primitive compared to the way things work in specs or other similar projects where you get an iterator with one dwarf component, one item_component, etc. I have a bit more boilerplate to write in order to get to the same effect, but I don't mind that right now.

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

Rust makes so easy to add tests that it's silly not to write some.
I have usually tests for bugs I find. Most of them are integration tests that test how various systems interact. Each time I find a bug I usually try and add a test for it. 

For example, when dwarves got hungry as they were carrying an item to craft something, they would somehow get that crafting task in a limbo state, abandoning it and going to eat a baguette. After I fixed it, it was trivial to add a test that checks tasks gets handled properly in that case.

Here is the test I added to check for that case:

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
    let hunger_limit = dwarf_component.get_stats().hunger_limit - 10;
    for _ in 0..hunger_limit {
        world.update(1);
    }

    //Create a task for him
    let task = PlaceItemTask::new(world.generate_entity_id(), Vec3::new(6, 6, 0), "Bed");
    world.add_task(Box::new(task));

    for _ in 0..250 {
        world.update(1);
    }

    assert!(world.get_entity_by_type("Baguette").is_none());
    assert!(world.get_entity_by_type("Bed").is_some());
}
```

## One year

![One year](/images/dwarf_game/first_commit.png)

I started this almost exactly one year ago and I commited changes to the project constantly. How do I manage to stay motivated for this long?

I have no good answer to this. It's a first for me too since I usually abandon side projects that take more than 2 months. This one kind of stuck with me. Maybe it's the fact that I really enjoy writing rust?

What I do know is that it's ok to take breaks. I've had periods where I had a lot of activity and pushed a lot of changes followed by a few weeks when I didn't write a single line of code. Last summer I did almost no work on it.

I also took breaks and worked on other projects, like [rust_hawktracer](https://github.com/AlexEne/rust_hawktracer).

Streaming on twitch also helps, even if I don't really have a regular schedule (I'm at episode 24 right now), it brings some sort of order and mini-deadline.

## What's next?

Things that I want to work on next are:
- Picking a name for it
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

