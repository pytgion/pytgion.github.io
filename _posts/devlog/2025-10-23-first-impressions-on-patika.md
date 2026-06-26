---
layout: post
title: "First Impressions on Patika"
date: 2025-10-23
category: devlog
---

When we started GermStorm, the concept was clear: a game about **fighting diseases in the human body**, blending RTS/Tower Defense mechanics with deck-building. We chose Godot over pretty much anything else—for obvious reasons—and began development. Halfway through, we hit a critical pathfinding problem: we couldn't update map obstacles in real time. It required re-computing paths constantly, which caused a performance catastrophe that I couldn't tolerate.

![GermStorm screenshot]({{ '/assets/images/ss_germstorm_0.png' | relative_url }})

This necessity spurred a **GDExtension** project to design a custom, decent pathfinding algorithm.

### Phase 1: The Object-Oriented Performance Wall

We are developing a 3D game with RTS mechanics on a Tower Defense framework. Although the map is 3D, we only compute paths on a single level, so a **2-dimensional matrix** should have been sufficient. Our initial C++ approach looked like this:

```cpp
std::vector<MapPoint> _map;
std::vector<CollisionObject> _obstacles;
std::vector<Agent> _agents;
```

What I missed was that in this scenario, every single agent needed to compute its way via Godot's bindings. This meant an enormous volume of **virtual calls** across the engine boundary. The result? Our profiling showed that **70% of the milliseconds per frame were caused by virtual calls**. I ended up with a miserable, impractical solution.

### Phase 2: Embracing C and Data-Oriented Design

My instinct was to drop to the metal. Refactoring the core library into **pure C** and utilizing **Structure of Arrays (SoA)** with raw pointers was the only way forward.

Once I committed to C, my inner low-level daemon took over. I now had a system built entirely for **data locality** and minimal overhead:

```c
typedef struct {
    AgentDataC* agents;
    int32_t agent_capacity;
    int32_t next_free_id;

    MapTileDataC* map_data;
    int32_t map_width;
    int32_t map_height;
    BarrackDataC* barrack_data;
    uint16_t barrack_capacity;
    uint16_t next_free_barrack_id;
} AgentManagerC;
```

It is incredibly robust and functions perfectly so far. The performance numbers have **dropped dramatically**, but we're not quite finished. We can now compute exactly how agents move (I'll detail the specifics soon), but the new challenge is solving **how to render that many agents** effectively.

### Phase 3: Instanced Rendering Hell

Once I took control at the low level, there was no turning back—passing every agent's data back to individual Godot objects would entirely defeat the purpose. My goal became clear: pass the raw agent data directly to the **GPU buffer**. To do this, I first needed a definitive, real-time method for finding the exact world position for every agent.

### Where to Handle Movement?

##### Option 1: Simulate movement in C core

The problem here is simple: it breaks the principle of my pathfinding library, **Patika**. The goal of Patika is strictly to calculate the _next step_ (the pathfinding logic). I could easily implement movement with delta time, but debugging complex motion and interpolation deep inside a C library called by a GDExtension would be a nightmare. I decided against ballooning the library's scope.

##### Option 2: Batch and track world_position data

This is the accepted compromise. I introduced a `MapGenerator` class (a subject for a future devlog!) that calculates and stores all necessary **world position data** for every hexagonal tile as a `std::vector<Vector3>`.

The `AgentManagerC` class then handles all agent movement, managing the interpolation between `agent.current_pos` (a hex index) and `agent.next_pos` (the destination hex index) via delta time.

The rendering solution leverages this structure: each `Barrack` object now holds a **MultiMeshInstance3D**. It collects all the raw position and transform data for its "own kind" of agents and fills its MultiMesh buffer. This caps the rendering cost at a maximum of **64 draw calls** (one per Barrack object), easily drawing thousands of agents.

#### Drawbacks: Data Separation and Complexity

This highly optimized approach does introduce a few architectural drawbacks:

- **Rendering Limitations:** Since all agents are batched in a single draw call per Barrack, applying advanced per-agent rendering effects like a **Fog of War** visibility test becomes significantly more complex (though this is not currently required by the Game Design Document).

- **Data Synchronization:** To render, each `Barrack` object needs to efficiently access and read the agents' computed positions from the central `AgentManagerC` based on agent IDs. This **data separation** makes initialization and agent deletion logic more complicated, as two distinct systems (the C core and the Godot rendering loop) must be perfectly synchronized.

### Conclusion

The low-level pivot was a success, shifting the performance bottleneck from the CPU to the GPU. Next time, I'll break down the **Map Generator's** hex-to-world coordinate system. Stay tuned!

..and in case I don't see ya, good afternoon, good evening, and good night!
