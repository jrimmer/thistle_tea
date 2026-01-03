# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Thistle Tea is a World of Warcraft 1.12.1 (vanilla) private server written in Elixir with Rust NIFs for pathfinding.

## Commands

### Build/Test/Lint
- `mix test` - Run all tests
- `mix test test/path/to/file_test.exs` - Run specific test file
- `mix test test/path/to/file_test.exs:123` - Run specific test at line 123
- `mix test.watch` - Run tests in watch mode
- `mix credo` - Run linting
- `mix format` - Format code

### Running
- `iex -S mix` - Start server interactively
- `GAME_SERVER=192.168.1.x iex -S mix` - Start with custom game server IP

### Setup
- `mix deps.get && mix deps.compile` - Install dependencies
- `cd assets && npm install` - Install frontend assets
- `./scripts/generate-mangos0-db.sh` - Generate world database (requires Docker)
- `./scripts/generate-dbc-db.sh` - Generate DBC database from client (requires WOW_DIR env var)
- `mix build_maps` - Generate navigation meshes (30+ minutes)

## Architecture

### Server Flow
1. **Auth Server** (`lib/auth/`) - Port 3724, handles SRP6 authentication
2. **Game Server** (`lib/game/network/server.ex`) - Port 8085, handles world packets

### Entity System
Entities use component-based composition in `lib/game/entity/`:
- **Data structs** (`data/`) - Pure data: `Mob`, `GameObject` compose components like `Object`, `Unit`, `MovementBlock`, `Internal`
- **Server GenServers** (`server/`) - `MobServer`, `GameObjectServer` manage entity lifecycle and message handling
- **Logic modules** (`logic/`) - Pure functions for movement, updates

### World Systems (`lib/game/world/`)
- **SpatialHash** - ETS-based spatial indexing for proximity queries (players, mobs, game_objects tables)
- **CellActivator** - Activates/deactivates world cells based on player proximity
- **Pathfinding** - Rust NIF wrapper for navigation mesh queries

### Network Protocol (`lib/game/network/`)
- **Opcodes** (`opcodes.ex`) - Complete opcode definitions
- **Message handlers** (`message/`) - One file per packet type (CMSG_*/SMSG_*)
- **Protocols** (`protocols.ex`) - `ClientMessage` and `ServerMessage` macros

### Databases
- **mangos0.sqlite** - World data (creatures, items, spawns) from MaNGOS
- **dbc.sqlite** - Client data (spells, races) extracted from WoW client
- **Mnesia** - In-memory accounts/characters (seeded on startup in `application.ex`)

## Code Style

### Network Messages
```elixir
# Server messages - implement to_binary/1
use ServerMessage, :SMSG_FOO

# Client messages - implement from_binary/1 and handle/2
use ClientMessage, :CMSG_BAR
```

### Entity Components
```elixir
# Pattern match on component structs
def nearby_players(%{internal: %Internal{map: map}, movement_block: %MovementBlock{position: {x, y, z, _o}}}, range) do
```

### Binary Parsing
```elixir
<<value::little-size(32), rest::binary>> = payload
```

### Formatting
- Max line length: 120 characters
- No comments in code
- Use `rescue` blocks in GenServer callbacks to prevent crashes
