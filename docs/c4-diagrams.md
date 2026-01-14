# Thistle Tea - C4 Architecture Diagrams

This document contains C4 model diagrams for the Thistle Tea WoW 1.12.1 server emulator.

## Level 1: System Context

Shows Thistle Tea in the context of its users and external systems.

```mermaid
C4Context
    title System Context Diagram - Thistle Tea

    Person(player, "Player", "A person playing World of Warcraft 1.12.1 client")
    Person(admin, "Administrator", "Server operator managing the game server")

    System(thistletea, "Thistle Tea", "WoW 1.12.1 vanilla server emulator written in Elixir")

    System_Ext(wowclient, "WoW Client", "World of Warcraft 1.12.1 game client (build 5875)")
    System_Ext(browser, "Web Browser", "Browser for admin dashboard access")

    Rel(player, wowclient, "Plays game using")
    Rel(wowclient, thistletea, "Connects via WoW protocol", "TCP 3724, 8085")
    Rel(admin, browser, "Manages server via")
    Rel(browser, thistletea, "Accesses dashboard", "HTTP 4000")
```

## Level 2: Container

Shows the major containers (processes/applications) within Thistle Tea.

```mermaid
C4Container
    title Container Diagram - Thistle Tea

    Person(player, "Player", "WoW 1.12.1 client user")
    Person(admin, "Administrator", "Server operator")

    System_Boundary(thistletea, "Thistle Tea") {
        Container(auth, "Auth Server", "Elixir/ThousandIsland", "Handles SRP authentication, session keys, and realm list")
        Container(game, "Game Server", "Elixir/ThousandIsland", "Core game logic: player connections, movement, combat, spells, AI")
        Container(web, "Web UI", "Elixir/Phoenix", "Admin dashboard, map visualization, LiveDashboard")
        Container(supervisor, "Entity Supervisor", "Elixir/DynamicSupervisor", "Spawns and manages player, mob, and game object processes")

        ContainerDb(memento, "Auth Database", "Memento (In-Memory)", "Accounts, characters, session data")
        ContainerDb(mangos, "World Database", "SQLite", "Creatures, items, NPCs, waypoints, game objects")
        ContainerDb(dbc, "Game Data", "SQLite", "Spells, races, classes from client DBC files")
        ContainerDb(ets, "Runtime State", "ETS Tables", "Sessions, spatial hash, GUID mappings")
    }

    Container_Ext(namigator, "Namigator", "Rust NIF", "Pathfinding via navigation meshes")

    Rel(player, auth, "Authenticates", "TCP 3724")
    Rel(player, game, "Plays game", "TCP 8085")
    Rel(admin, web, "Manages", "HTTP 4000")

    Rel(auth, memento, "Validates credentials")
    Rel(auth, ets, "Stores session keys")
    Rel(game, ets, "Reads sessions, spatial data")
    Rel(game, supervisor, "Spawns entities")
    Rel(game, mangos, "Loads world data")
    Rel(game, dbc, "Reads game definitions")
    Rel(game, namigator, "Queries paths")
    Rel(supervisor, memento, "Loads characters")
```

## Level 3: Component - Game Server

Shows the internal components of the Game Server container.

```mermaid
C4Component
    title Component Diagram - Game Server

    Container_Boundary(game, "Game Server") {
        Component(network, "Network Layer", "ThousandIsland Handler", "Manages TCP connections, packet encryption/decryption")
        Component(packet, "Packet Dispatcher", "Elixir Module", "Routes CMSG opcodes to appropriate message handlers")
        Component(messages, "Message Handlers", "Elixir Modules", "50+ handlers for client/server messages (CMSG/SMSG)")

        Component(entities, "Entity System", "GenServers", "Individual processes for players, mobs, game objects")
        Component(components, "Entity Components", "Elixir Structs", "Object, Unit, Player, MovementBlock, Internal data")
        Component(logic, "Entity Logic", "Pure Functions", "Core operations: damage, healing, positioning")
        Component(ai, "AI System", "Behavior Trees", "Mob AI with Selector/Sequence/Action nodes")

        Component(spatial, "Spatial Hash", "ETS Tables", "Cell-based proximity queries for nearby entities")
        Component(cellact, "Cell Activator", "GenServer", "Loads/unloads entities based on player proximity")
        Component(loader, "Entity Loader", "Elixir Modules", "Loads mobs and objects from database by cell")
        Component(events, "Game Events", "GenServer + PubSub", "Time-based world events and broadcasts")

        Component(pathfinding, "Pathfinding", "NIF Interface", "Queries Namigator for navigation mesh paths")
    }

    ContainerDb_Ext(ets, "ETS Tables", "Runtime state")
    ContainerDb_Ext(mangos, "Mangos DB", "World data")
    Container_Ext(namigator, "Namigator", "Rust pathfinding")

    Rel(network, packet, "Decrypted packets")
    Rel(packet, messages, "Dispatches by opcode")
    Rel(messages, entities, "Updates entity state")
    Rel(messages, spatial, "Queries nearby entities")

    Rel(entities, components, "Composed of")
    Rel(entities, logic, "Calls for operations")
    Rel(entities, ai, "Executes behavior")
    Rel(ai, pathfinding, "Requests paths")

    Rel(spatial, ets, "Stores/queries")
    Rel(cellact, spatial, "Monitors player positions")
    Rel(cellact, loader, "Triggers loading")
    Rel(loader, mangos, "Queries entities")
    Rel(loader, entities, "Spawns via supervisor")

    Rel(events, entities, "Broadcasts updates")
    Rel(pathfinding, namigator, "NIF calls")
```

## Level 3: Component - Entity System Detail

Shows the entity component architecture in more detail.

```mermaid
C4Component
    title Component Diagram - Entity System

    Container_Boundary(entity_system, "Entity System") {
        Component(player_srv, "Player Handler", "Network.Server", "Per-connection process handling player state and packets")
        Component(mob_srv, "Mob Server", "Entity.Server.Mob", "Per-mob GenServer with AI tick loop")
        Component(obj_srv, "GameObject Server", "Entity.Server.GameObject", "Static/interactive world objects")

        Component(obj_comp, "Object Component", "Data.Component.Object", "GUID, entry ID, type, scale")
        Component(unit_comp, "Unit Component", "Data.Component.Unit", "Health, mana, level, stats, faction")
        Component(player_comp, "Player Component", "Data.Component.Player", "Race, class, gender, player-specific data")
        Component(move_comp, "MovementBlock", "Data.Component.MovementBlock", "Position, orientation, movement flags")
        Component(internal_comp, "Internal Component", "Data.Component.Internal", "Behavior tree, name, map ID")

        Component(core_logic, "Core Logic", "Entity.Logic.Core", "Damage, healing, death, stats calculation")
        Component(move_logic, "Movement Logic", "Entity.Logic.Movement", "Waypoints, pathfinding, position sync")
        Component(bt_logic, "Behavior Tree", "Entity.Logic.AI.BT", "BT execution: tick, node traversal")

        Component(update_mask, "Update Mask", "Entity.UpdateMask", "Efficient binary field serialization")
    }

    Rel(player_srv, obj_comp, "Has")
    Rel(player_srv, unit_comp, "Has")
    Rel(player_srv, player_comp, "Has")
    Rel(player_srv, move_comp, "Has")

    Rel(mob_srv, obj_comp, "Has")
    Rel(mob_srv, unit_comp, "Has")
    Rel(mob_srv, move_comp, "Has")
    Rel(mob_srv, internal_comp, "Has")
    Rel(mob_srv, bt_logic, "Executes")

    Rel(obj_srv, obj_comp, "Has")
    Rel(obj_srv, move_comp, "Has")

    Rel(mob_srv, core_logic, "Calls")
    Rel(mob_srv, move_logic, "Calls")
    Rel(player_srv, core_logic, "Calls")
    Rel(player_srv, update_mask, "Serializes with")
    Rel(mob_srv, update_mask, "Serializes with")
```

## Data Flow Diagrams

### Player Login Flow

```mermaid
sequenceDiagram
    participant C as WoW Client
    participant A as Auth Server
    participant S as Session (ETS)
    participant G as Game Server
    participant E as Entity Supervisor
    participant H as Spatial Hash

    C->>A: AUTH_LOGON_CHALLENGE
    A->>A: Generate SRP challenge
    A-->>C: Challenge response

    C->>A: AUTH_LOGON_PROOF
    A->>A: Verify SRP proof
    A->>S: Store session key
    A-->>C: Proof response + realm list

    C->>G: CMSG_AUTH_SESSION
    G->>S: Validate session key
    G-->>C: SMSG_AUTH_RESPONSE

    C->>G: CMSG_PLAYER_LOGIN
    G->>E: Spawn player GenServer
    E-->>G: Player PID
    G->>H: Register player position
    G-->>C: SMSG_LOGIN_VERIFY_WORLD
    G-->>C: Initial world state packets
```

### Mob AI Tick Flow

```mermaid
sequenceDiagram
    participant M as Mob GenServer
    participant BT as Behavior Tree
    participant L as Logic.Core
    participant H as Spatial Hash
    participant P as Nearby Players

    loop Every 100ms
        M->>BT: tick(behavior_tree, blackboard)
        BT->>BT: Evaluate nodes
        BT-->>M: {status, new_state}

        alt Combat State
            M->>H: Query nearby enemies
            H-->>M: Target list
            M->>L: Calculate damage
            L-->>M: Damage result
        else Patrol State
            M->>M: Follow waypoints
        end

        M->>H: Update position
        M->>P: Broadcast SMSG_UPDATE_OBJECT
    end
```

### Cell Activation Flow

```mermaid
sequenceDiagram
    participant CA as Cell Activator
    participant H as Spatial Hash
    participant L as Entity Loader
    participant DB as Mangos DB
    participant S as Entity Supervisor

    loop Every 1 second
        CA->>H: Query all player positions
        H-->>CA: Player position list

        CA->>CA: Calculate active cells (with adjacency)
        CA->>CA: Diff with current active cells

        alt New cells to activate
            CA->>L: load_cell(map, cell_x, cell_y)
            L->>DB: Query creatures in cell
            DB-->>L: Creature list
            L->>DB: Query game objects in cell
            DB-->>L: GameObject list

            loop Each entity
                L->>S: start_child(entity_spec)
                S-->>L: {:ok, pid}
            end
        end

        alt Cells to deactivate
            CA->>S: Terminate entities in cell
        end
    end
```
