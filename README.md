# multiplayer-fabric-skills

Skills library for V-Sekai multiplayer-fabric agents.

A skill is a self-contained, reusable behaviour unit that zone_console,
zone backend workers, or Godot zone processes can call to accomplish a
discrete task (e.g., upload an asset, instance a scene, query a shard list).

## Structure

Skills live in `skills/`. Each file is standalone with no cross-skill imports.

## Usage

Add as a dependency in `mix.exs`:

```elixir
{:multiplayer_fabric_skills, github: "V-Sekai-fire/multiplayer-fabric-skills"}
```
