---
layout: post
comments: true
---

> I wish today's games would have accurate temperature simulation...

I hear that all the time.   
Look no further, if you want to make your game more true to reality in terms of how temperature works I have all you need to get you bootstrapped.  
We already have physically-based rendering, so why did we stop there?!

## But... Why?

Because it's my game and I do whatever I want.  
I am working on a sim game where you have to take care of a colony of dwarves and I want seasons to play a big part in this.  
I want my dwarves to be cold in the winter, get warm in the summer, lose a limb due to frostbites. Many games have already experimented with such systems to some degree (e.g. Don't Starve).

__But can't that be solved with a simple radius check and a decay formula for the temperature transferred?__   
Yes you can and you should do that, but for me that wasn't enough for a few reasons:  
1) I wanted to have cave temperatures be slowly influenced by  the outside temperatures. E.g. you can grow mushrooms in your cave but not near the cave entrance as they might need a cooler and more stable temperature.  
2) I want certain plants and foods to preserve better at lower temperatures.  
3) Another thing that complicated my life are walls. If you have a fire in one room near a wall you don't heat the air outside of that room at the same rate as you heat the room.  

## The solution
In addition to the reasons mentioned above it is really hard to tune random formulas that have nothing to do with reality. It's way easier to start from a realistic system and formulas and tune that to make it fun.

Let's go to the classic formula of calculating the heat transferred when a system goes from one temperature to another.

<div>
$$
Q = m * c * {\Delta}T  
$$
</div>

`Q` = Heat, measured in Joules - `J`   
`m` = mass of the system grams - `g`  
`c` = specific heat capacity of a substance. That is the energy required (heat) such that a unit of mass (g) of that substance will raise its temperature by one unit (Kelvins - `K`).  

So, as far as units of measurement are, we have an equation of the form:  

<div>
$$
J = g * \dfrac{J}{g*K} * K
$$
</div>

So it all checks out and we are left with `Q` being measured in Joules.

But just that formula is not enough as it doesn't tell us how two adjacent terrain cells at different temperatures should transfer heat between each other.  
When two bodies have different temperatures, their molecules have different average kinetic energies (they bounce around at different speeds). When these two bodies are in contact, collisions between moving molecules on the surface of contact will transfer energy from the high-temperature body to the low-temperature one.  
As that transfer happens, the higher temperature body slightly cools down and the lower temperature body warms up.  
We intuitively already know that different materials conduct energy at different rates. That's why you have air in between two sheets of glass in your windows. Air is a material that has a high thermal resistivity.

If you research this online you find these material constants defined as either `thermal resistivity` or `thermal conductivity`. They are the basically expressing the same thing, but you divide by one and multiply by the other.

For thermal conductivity we have this formula:
<div>
$$
\dfrac{Q}{t} = \dfrac{(k * A * {\Delta}T)}{d}
$$
</div>

Where:  
`t` = time    
`Q` = heat  
`A` = surface area  
`T` = temperature  
`d` = distance to the point we want to measure the temperature at.  
`k` = thermal conductivity  

For thermal resistivity the formula will look like:
<div>
$$
\dfrac{Q}{t} = \dfrac{A*{\Delta}T}{resistivity * d} 
$$
</div>

I went with the thermal resistivity expression as engineering books seem to favor it and they provide constants for a variety of building materials I have in my game. If you find conductivity values, you can easily switch from one to the other. __Just make sure your units of measurement work out correctly. If you express everything in SI units you will be fine.__

## Putting it all together

The terrain in my game is a 3-D grid of cells (think minecraft-like). Some cells are made of ground, some are air, stone, etc.  
That being said, update cycle that happens for temperatures in my game is made up of the following steps.

### 0 - Heat propagation from the "sun"
All terrain cells that _see_ the sky will get their temperatures updated based on some ambient temperature based on the time of day and season. 
All the other terrain cells will tend to get towards a different ambient temperature based on how deep they are, etc.
This step is here to simulate a fake convection and radiation. Without it all the map will eventually heat up from a fireplace (as there would be just energy added, and none lost).  
It's also very easy to tune seasons temperatures like this so that's why nothing complex is going on this step.

### 1 - Heat propagation for environment (terrain cells)
For this we apply this formula as we need to know `Q`, the heat transferred between two adjacent terrain cells.  

<div>
$$
{Q} = \dfrac{A*t*{\Delta}T}{resistivity * d} 
$$
</div>

Using the formula above we compute the average energy a terrain cell is transferring with its neighbors.
I consider a cell's temperature as the temperature measured at the center of that cell.
This energy is used to compute a cell's new temperature by plugging `Q` in this equation:

<div>
$$
T_Final = T_Initial + \dfrac{Q}{m*c} 
$$
</div>

We have the final temperature of our terrain cell.  
You can see how this process is super easy to parallelize, as each cell updates it's temperature from the previous temperature of the neighbor cells. You just need to have two terrain copies (one with the old temperatures, and one that will get the new temperatures based on the old values).

### 2 - Heat propagation to game entities

For this we use the same formulas as above, with a small exception. We just update the game entity's temperature and we don't change the terrain cell temperature based on that game entity. This is for _receiving_ game units (dwarves, objects, cows, etc.). These entities have a MaterialComponent attached to them. That component tracks the temperature and other useful material properties to compute that temperature. Applying the proper formula doesn't yeld nice results due to the unrealistic terrain cell to dwarf size ratio (see bottom of the article).  
`Emitting` game units like campfires propagate the temperature in the other direction, from them to the terrain.

This is the definition of the `MaterialComponent`:
```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct MaterialComponent {
    volume: f32,
    area: f32,
    temperature: f32,
    radius: f32,
    mass: f32,
    material: Material,
}
```

Let's remember the properties needed to transfer temperature between two entities:
* Surface area
* Initial_temperature
* Distance (that the energy _travels_)
* Specific specific heat capacity
* Thermal resistivity
* Time
* Mass

From that list, just a few are entity-specific: __surface area__, __distance__, __mass__.  
Besides time, the rest depend on the material so we can keep them in a separate `material_properties.json` table.  
That is a json to make it easy to add and tune materials.

__But how do we know the mass, surface area and distance that the heat travels for a dwarf body part?__

Configuring the __distance energy travels__, __mass__ and __surface area__ for each body part component is an interesting exercise, but I am lazy and here I went with an approximation with the goal of keeping things intuitive and easy to set-up.  

The only parameters actually needed to initialize the `MaterialComponent` for an entity (head, hand, leg, etc.) are: 

```json
{
    "MaterialComponent": {
        "mass": 6000,
        "material": "Water"
    }
}
```  

Besides the temperature-related `material_properties.json` entry, I also input the density in that table.
For example the entry for `"Water"` is:

```json
"Water": {
    "density": 997000.0,
    "thermal_resistivity": 1.6,
    "specific_heat_capacity": 4.2
}
```

You probably know where this is going.  
__For the purpose of temperature transfer, I am approximating all entities to simple shapes (spheres).__

That's where the density comes into play. With it and the mass we can compute the volume of our entity:

<div>
$$
Volume = \dfrac{Mass}{Density}
$$
</div>

Having the `Volume`, using a simple formula we can compute the `radius` of the sphere and from that the `area`.  
We now have all the data needed, so we do the same calculations using the same formulas we used for the terrain temperature.  
All `MaterialComponents` can be updated in parallel.

### 3. Needs system

The needs system reads the temperature values from the `MaterialComponent` and issues Actions based on the needs of a creature(dwarf, cow, etc.). E.g. If they are cold they go near the fire. I will write more about the needs system and how AI works in a different blog post.

## Next steps

The next steps is to take armor and clothes into account when calculating the heat transferred. That just slightly changes the formula we have as we just need to add the R-values of all materials involved in the heat transfer:

<div>
$$
R_value = thermal_resistivity * distance
Total_R_Value = Rvalue_of_entity_1 + RValue_of_entity_2 + ...
$$
</div>

Using spheres will make it a bit tricky to handle clothes as the clothes would need to be hollow and I will probably consider all entities with `MaterialComponents` cube-shaped instead.  
I plan on adding more temperature related effects.  
Conduction is just one way heat is transferred between things. Another way to exchange heat is convection and radiation. I am not sure if I will add them as the effects in this game world will probably be so small it's not worth it.  

Because you reached the end of this, I leave you with this cozy picture of a dwarf and a cow warming up near a fire.

![dwarf_and_cow_by_the_fire](/images/temperature/dwarf_and_cow.png)
