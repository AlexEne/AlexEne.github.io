---
layout: post
comments: true
---

> I wish today's games would have accurate temperature simulation...

I hear that all the time.   
Look no further, if you want to make your game more true to reality in terms of how temperature works I have all you need to get you bootstrapped.  
We already have phisically-based rendering, so why stop there?!

## But...Why?

Because it's my game and I do whatever I want.  
I am working on a 2D simulation game where you have to take care of a colony of dwarves. I also want seasons to play a big part in this survival game. I want my dwarves to be cold in the winter, get warm in the summer, lose a limb due to frostbites. Many games have already experimented with such systems to some degree (e.g. Don't Starve)

__But can't that be solved with a simple radius check and a decay formula for the quantity of heat transferred?__   
Yes you can and you should do that, but for me that wasn't enough for a few reasons.
1) I wanted to have cave temperatures be slowly influenced by  the outside temperatures. E.g. you can grow mushrooms in your cave, but not near the enterance as they might need a slightly colder environment.
2) Another thing that complicated my life are walls. If you have a fire in one room near a wall you don't heat the outside of that room at the same rate as you heat the room.

## The solution
I did start with a simple distance check but once walls came into play I paused and started to think about this problem thoroughly.  
Also in addition to that it is really hard to tune random invented formulas that have nothing to do with reality. And this is an important point that I will come back to later.

So I started refreshing my high-school thermodynamics formulas and terms thinking there should be something in there to help me solve my problem with temperature transfer between terrain cells.

And we get to the classic formula of calculating the heat transferred when a system goes from one temperature to another:

<div>
$$
Q = m * c * {\Delta}T  
$$
</div>

```
Q = m × c × ΔT

Q = Heat, measured in J (Joules)
m = mass of the system (g)
c = specific heat capacity of a substance. That is the energy required (heat) such that a unit of mass (g) of that substance will raise its temperature by one unit (of temperature - K).
```

So, as far as units of measurement are, we have an equation of the form:  

<div>
$$
J = g * \dfrac{J}{g*K} * K
$$
</div>

So it all checks out and we are left with Q being measured in Joules.

But what we are interested in is how the final temperature evolves given we know the heat that we are receiving on a terrain cell?  
In order to know that, we need to leaverage another simple formula for Conductivity.
When two bodies have different temperatures, their molecules have different average kinetic energies (they bounce around at different speeds). When these two bodies are in contact, collisions on the surface of contact will transfer energy from the high-temperature body to the low-temperature one.  
As that transfer happens, the higher temperature body slightly cools down and the lower temperature body warms up.  
We intuitively already know that different materials conduct energy at different rates. That's why you have air in between two sheets of glass in  your home. Air is a material that has a high thermal resistivity.

Now, if you research this online you find these rates defined as either thermal resistivity or thermal conductivity. They are the same thing, but you multiply by one and divide by the other.

For thermal conductivity (k below is thermal conductivity):
<div>
$$
\dfrac{Q}{t} = \dfrac{(k * A * {\Delta}T)}{d}
$$
</div>
Where:

t = time  
Q = heat
A = surface area
T = temperature
d = distance to the point we want to measure the temperature at.
k = thermal conductivity

For thermal resistivity the formula will look like:
<div>
$$
\dfrac{Q}{t} = \dfrac{A*{\Delta}T}{resistivity * d} 
$$
</div>

I went with the thermal resistivity expression as engineering books seem to favor it and provide constants for a variety of materials I needed. If you find conductivity, you can easily switch from one to the other. Just make sure your units of measurement work out correctly. If you express everything in SI units you will be fine.

## Putting it together

The terrain in my game is a 3-D grid of cells, some cells are made of ground, some are air, some are stone, etc.  
The update cycle that happens for updating temperatures in my game is made up of the following steps.

### 0 - Heat propagation from the "sun".
All terrain cells that _see_ the sun will get their temperatures updated based on some ambient temperature. This is  here to simulate a fake convection and radiation as without it all the map will eventually heat up from a fireplace (as there would be just energy added, and none lost).  

### 1 - Heat propagation for terrain cells. 
For this we apply this formula as we need to know Q (the heat transferred).  

<div>
$$
{Q} = \dfrac{A*t*{\Delta}T}{resistivity * d} 
$$
</div>

This energy is then used to compute a cell's new temperature by plugging Q in the first equation:

<div>
T_Final = T_Initial + \dfrac{Q}{m*c} 
</div>

After this we have the final temperature of our terrain cells.  
You can see how this process is super easy to parallelize, as each cell updates it's temperature from the previous temperature and the neighbour cells temperature. You just need to have two terrain versions (one with the old temperatures, and one that will get the new temperatures based on teh old values).

### 2 - Heat propagation to game units

For this we use the same units as above, with a small exception, we just update the game unit's temperature and we don't change the terrain cell temperature based on that game unit. This is for receiving game units (dwarves, objects, cows, etc.). These entities have a material component attached to them. That component tracks the temperature and other useful material properties to compute that temperature.

This is the definition of the `MaterialComponent`:
```
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

__But how do we know the mass and distance that the heat travels for a dwarf body part?__

Now let's remember the properties needed to transfer temperature between two entities:
* Surface area
* Initial_temperature
* Distance (that the energy _travels_)
* Specific specific heat capacity
* Thermal resistivity
* Time
* Mass

From that list, a few are entity-specific: __surface area__, __distance__, __mass__.  
Besides time, the rest depend on the material so we can keep them in a separate `material_properties.json` table.  
That is still a json to make it easy to add new materials.

Configuring the __distance__ and __mass__ and __surface area__ for each body part component is an interesting exercise, but I am lazy and here I went with an approximation and kept things intuitive and easy to set-up.  

The only parameters actually needed to initialize the material component are: 

```json
{
    "Material": {
        "mass": 6000,
        "material": "Water"
    }
}
```  
These are specified in the recipe table as input.  
And here is the simple trick I did. Besides the temperature-related material properties, I also input the density in that table.
For example the material_properties entry for `"Water"` is:

```json
"Water": {
    "density": 997000.0,
    "thermal_resistivity": 1.6,
    "specific_heat_capacity": 4.2
}
```

You probably know where this is going. For the purpose of temperature transfer, I am approximating all entities to spheres.
That's where the density comes into play, as with it and the mass we can compute the volume of our entity:


<div>
$$
Volume = \fracd{Mass}{Density}
$$
</div>

Then using simple trigonometry we get the surface area of a sphere and the radius. 
As we now have all the data needed, we do the same calculations using the same formulas used for the terrain.

### 3. Needs system

The needs system reads the temperature values from the Material component and issues Actions based on the creature(dwarf, cow, etc.) needs. E.g. If cold they go near the fire. I will write more about the AI in a different blog post.

## Next steps

### Clothing
The next steps is to take clothing into account when calculating the heat transfer. That just slightly changes the formula we have as we just need to add the R-values of all materials involved in the heat transfer:

<div>
$$
R_value = thermal_resistivity * distance
Total_R_Value = Rvalue_of_entity_1 + RValue_of_entity_2 + ...
$$
</div>

Spheres here will make it a bit tricky and I will probably switch to considering them cubes instead.  
In addition to that I plan on adding more effects induced by cold besides freezing.  
Conduction is just one way heat is transferred between things. A nice and not too hard addition to that is convection. I am not sure if I will add it as the effects in this game world will probably be so small it's not worth it.  

And in the end I leave you with this picture of a dwarf and a cow warming up near a fire.

![dwarf_and_cow_by_the_fire](/images/temperature/dwarf_and_cow.png)
