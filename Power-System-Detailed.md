# Power System Plugin Overview

## Core Goal  
Manage a flexible, performant electricity network system with producers, consumers, batteries (storage), switches, and hierarchical networks/grids—supporting priority-based load balancing, battery charge/discharge logic, event-driven updates, dynamic grid isolation, and throttling/load shedding.

Updates are planned to run asynchronously in background threads, with queueing of work to prevent blocking the game thread. Multithreaded execution of balancing and aggregation will boost scalability and responsiveness.

Tick and update frequencies are separated to optimize CPU usage by throttling low-priority or stable networks.

---

## 1. System Structure

### 1.1 World Subsystem: `UPowerSystemSubsystem`  
- Singleton managing all power networks in the world.  
- Maintains and updates all active `UPowerNetwork` instances.  
- Coordinates global ticking, background queueing of update tasks, and event dispatching.  
- Manages synchronization between async updates and game thread state.  
- Responsible for scheduling network recalculations asynchronously, batching dirty updates efficiently.

### 1.2 Network Classes

| Class           | Role                                                              |
|-----------------|-------------------------------------------------------------------|
| `UPowerNetwork` | Top-level controller managing multiple connected grids. Balances all producers and batteries globally across grids. Detects isolated clusters and balances them independently in the background. Contains a unique `GridID` to track instances and for debugging or serialization. |
| `UPowerGrid`    | Logical contained power domain holding producers, consumers, and storage devices. Reports demand and status to `UPowerNetwork`. Performs local balancing and load shedding based on priority and throttling flags. Supports configurable update intervals independent from the main tick rate. |

---

## 2. Device Components

Each device type derives from a base class `UPowerDeviceComponent` which defines virtual methods for power management, replacing interface overhead for better performance and type safety.

| Component                  | Description                                                     |
|----------------------------|-----------------------------------------------------------------|
| `UPowerProducerComponent`  | Supplies power with max output, priority (integer), and dynamic output control. Registers to its owning `UPowerGrid`. Implements virtual functions to report current output and adjust production. |
| `UPowerConsumerComponent`  | Requests power with load and integer priority. Includes `bAllowThrottling` boolean to enable load shedding. When power is insufficient, consumers with `bAllowThrottling = false` disable first, shedding load without drawing power. Remaining power is distributed among consumers with throttling enabled, favoring those connected earlier. |
| `UPowerStorageComponent`   | Manages stored energy (batteries, capacitors), charge/discharge rates, capacity, and integer priority. Registers to `UPowerGrid`. Implements virtual functions for state updates and charge/discharge behavior respecting priority schemes. |
| `UPowerSwitchComponent`    | Controls power flow on/off within or between grids. Acts as breakers or isolators. Isolated subnets dynamically detected by the network. |
| `UPowerRelayComponent`     | Passive wiring/distribution box component. Devices connected through it belong to the same grid without limiting power flow. |

Devices register/unregister with their owning `UPowerGrid` during lifecycle events (begin/end play). All device power state queries and updates are routed through their base class virtual methods.

---

## 3. Interfaces and Communication

The system minimizes use of Unreal Interfaces in favor of a common base class with virtual methods for devices to improve performance.

### 3.1 Communication Structures

Upward reporting and downward budgeting use well-defined data structures to clarify flow and extendability:

| Name               | Type                  | Description                                                        |
|--------------------|-----------------------|--------------------------------------------------------------------|
| `FPowerDemandReport`| Struct                | Aggregated demand, battery status, priority info reported from grids/subgrids upward to `UPowerNetwork`. Includes load sums, throttling info, and grid state flags. |
| `FPowerBudget`      | Struct                | Power budgets and production targets issued downward from `UPowerNetwork` to grids/subgrids, containing allocated power per priority group and throttling instructions. |

### 3.2 Load Shedding and Throttling Logic

- Power is distributed in strict order of priority groups (higher integer = higher priority).  
- Within each priority group, devices are sorted by connection order; earlier connected devices receive power first.  
- For each priority group:

  1. Devices with `bAllowThrottling = false` are powered only if sufficient power exists to supply them fully; otherwise, they are disabled without drawing power (load shed first).  
  2. Remaining power is distributed among devices with `bAllowThrottling = true`, allowing partial or throttled power draw.  
  3. If power remains after fully powering all devices with throttling enabled, leftover power can then be allocated to previously skipped `bAllowThrottling = false` devices, but **only if the device can be powered fully**. Partial power is never assigned to these.  
  4. When no more devices in the current priority group can receive power, the algorithm proceeds to the next lower priority group, repeating the process.  

This logic ensures critical devices that cannot throttle get priority shutdown first under constrained power, while devices that can throttle share power proportionally, maximizing total usable output without partial starvation of unthrottled devices.


### 3.3 Update Scheduling and Frequencies

- Networks and grids support configurable update intervals independent from the main game tick rate, allowing throttling of expensive recalculations on less critical systems.  
- Dirty state changes on devices trigger scheduled update tasks that are queued for asynchronous execution.  
- Background threads process queued updates for balancing, load shedding, and aggregation to minimize main thread CPU impact.  
- Stable parts of the network can skip updates until device state changes.  
- Forced full updates or fallback periodic syncing ensure system consistency.

---

## 4. Priority & Load Balancing

Power distribution is handled through a **single-pass, priority-based algorithm**. Devices are grouped by integer priority values, and power is distributed starting from the highest priority group to the lowest.

### Priority Levels

Priority is represented as an integer value assigned to each device. Higher numbers represent higher priority.  
Example levels:

- 100 — Critical systems  
- 75 — High priority  
- 50 — Medium priority  
- 25 — Low priority  
- 0 — Lowest priority

This allows flexible, fine-grained tuning of power distribution across devices.

### Priority Balancing Rules

1. **Sort by Priority Group**  
   - Devices are grouped by their `Priority` integer.
   - Higher integers receive power before lower ones.

2. **Order by Connection**  
   - Within each group, devices are ordered by connection time (earlier connected = higher precedence).

3. **Throttling & Load Shedding Logic**  
   For each priority group:
   - **Step 1: Non-Throttle Devices (`bAllowThrottling = false`)**
     - Devices are only powered if enough supply exists to meet their full demand.
     - If not, they are **disabled and draw no power** (load shed).
   - **Step 2: Throttle-Capable Devices (`bAllowThrottling = true`)**
     - Remaining power is divided among these, allowing partial power.
   - **Step 3: Recheck Non-Throttle Devices**
     - If power remains, attempt to fully power any unpowered non-throttle devices from Step 1.
   - **Step 4: Move to Next Group**
     - If no further devices in the current group can be powered, proceed to the next lower group.

4. **Battery Integration**
   - Batteries are charged only from **surplus** after all active loads are met.
   - Batteries with higher priority (e.g., mission-critical backups) are filled first.
   - During shortages, high-priority batteries may **discharge to support active loads**.

5. **Resulting Budget**
   - After calculations, each `UPowerGrid` receives an `FPowerBudget` struct defining how much power is available per device, along with instructions for throttling.

---

## 5. Connection & Topology

The power system is flexible and spatially aware, supporting custom topologies via wires, relays, and switches.

### 5.1 Connection Structure

| Name              | Type                   | Description                                               |
|-------------------|------------------------|-----------------------------------------------------------|
| `ComponentA`      | `TWeakObjectPtr<UActorComponent>` | First connected component with power logic              |
| `ComponentB`      | `TWeakObjectPtr<UActorComponent>` | Second connected component                                |
| `SlotA`           | `FName`                | Wire attachment point on `ComponentA`                     |
| `SlotB`           | `FName`                | Wire attachment point on `ComponentB`                     |
| `ConnectionObject`| `TSoftObjectPtr<AActor>` | Optional wire actor for visual representation (e.g., spline or cable) |

### 5.2 Connection Rules

- **Relay Connections (`UPowerRelayComponent`)**
  - Devices connected through relays belong to the **same `UPowerGrid`**.
  - Power flows freely and passively without logic gates.

- **Switch Connections (`UPowerSwitchComponent`)**
  - Devices connected across a switch belong to **separate grids**.
  - Switch state determines whether grids are connected or isolated.

- **Topology Changes**
  - When a switch opens or closes, the system re-evaluates network structure.
  - Isolated grids are automatically split and balanced independently by `UPowerNetwork`.

### 5.3 Grid Identification

| Field     | Type     | Description                                  |
|-----------|----------|----------------------------------------------|
| `GridID`  | `FGuid`  | Unique identifier per `UPowerGrid` or `UPowerNetwork`. Used for debugging, save/load, and traceability. |

---

## 6. Event-Driven Updates

To maximize performance, the system avoids constant per-frame updates and instead responds only to meaningful changes.

### 6.1 Dirty State Propagation

- Devices track last known power state (output, load, battery level).
- When a device changes state, it marks its `UPowerGrid` as **dirty**.
- The grid queues an update task with `UPowerNetwork`, which decides **when and how** to process it (based on priority, frequency, and background queue availability).

### 6.2 Update Triggers

Updates can be triggered by:

- Change in device demand, output, or battery state
- Connection/disconnection of components
- Grid topology changes (e.g., switch toggled)
- External systems requesting recalculations

### 6.3 Scheduling

- Dirty grids are scheduled for updates based on their individual `UpdateFrequency` (in Hz or interval seconds).
- All updates are executed **asynchronously** when possible using a background queue.
- After updates complete, results are applied to the game thread on a deferred tick or task sync point.

### 6.4 Stable Skipping

- Grids with no dirty flags are skipped.
- Devices that haven't changed for a configurable duration may opt-out of updates entirely until re-activated.

### 6.5 Manual Controls

- Developers may force:
  - Full system recalculation
  - Targeted update of a specific network or device
  - Periodic sync mode fallback for non-event-based systems

---

## 7. Additional Gameplay Features

The power system supports various gameplay and simulation mechanics to allow full control over grid behavior, maintenance, and design.

### 7.1 Grid Enable/Disable

- Individual `UPowerGrid` instances can be temporarily disabled:
  - Disables all power flow within the grid.
  - Useful for scripted maintenance, power outages, or sabotage scenarios.

### 7.2 Switch Logic and Automation

- `UPowerSwitchComponent` can be toggled manually or through scripted logic:
  - Acts as circuit breakers or intelligent routing gates.
  - Can be controlled through gameplay triggers, logic blueprints, or remote systems (e.g., control panels, automation AIs).

### 7.3 Relay Components

- `UPowerRelayComponent` acts as a logical connection (like a wire or power bus).
  - Devices connected through a relay are treated as part of the same physical grid.
  - Has **no logic**, no limits—just defines connectivity.

### 7.4 Complex Priority Schemes

- Devices can be given **custom integer priority** values to create nuanced behavior:
  - Emergency batteries (priority 100) only discharge when all others are exhausted.
  - Background systems (priority 10) shut down quickly in emergencies.
  - Supports fine-tuned control over production, consumption, and storage.

### 7.5 Visual Wiring

- Visual representations of power links are supported through `ConnectionObject` actors:
  - Can be splines, dynamic cables, or blueprinted visuals.
  - Useful for debugging, gameplay immersion, and construction tools.
  - Automatically hide/show or colorize based on power status or debug overlays.

---

## 8. Communication Flow

Power flow occurs in a **bi-directional hierarchy** between devices, grids, and the network controller.

### 8.1 Upward Reporting

- `UPowerGrid` aggregates status from all producers, consumers, and storage devices.
- Builds an `FPowerDemandReport`:
  - Total load per priority group
  - Number of throttle-capable consumers
  - Available and reserved capacity
  - Device counts, charge levels, output caps

- This report is passed to the `UPowerNetwork`, which uses it to:
  - Plan power distribution
  - Decide on battery discharge
  - Calculate budgets and priorities

### 8.2 Downward Budgeting

- After processing, the network sends each grid an `FPowerBudget`:
  - Assigned production targets
  - Power per priority group
  - Battery charge/discharge permissions
  - Throttling instructions

### 8.3 Internal Grid Flow

Each `UPowerGrid`:

- Sorts consumers by priority
- Applies throttling rules (see Section 4)
- Distributes power accordingly
- Reports unmet demand or overflow
- Handles battery behavior as instructed

### 8.4 Dirty Event Propagation

| Direction  | Path                            | Purpose                                               |
|------------|----------------------------------|-------------------------------------------------------|
| Device → Grid | Mark grid dirty if load/output/state changes | Triggers recalculation at grid level                 |
| Grid → Network | Aggregate and escalate demand/status       | Network decides global allocation                    |
| Network → Grid | Push updated budget and instructions       | Grid rebalances its devices accordingly              |
| Async → Game Thread | Final update application               | Ensures safe and smooth integration of async results |

---

## 9. Performance & Scalability

The power system is built for scalability and responsiveness, suitable for large simulation-heavy worlds.

### 9.1 Event-Driven Logic

- No constant polling—only changes trigger updates.
- Drastically reduces CPU usage in stable networks.

### 9.2 Update Queue

- Dirty grids are placed into a **background task queue**.
- Tasks are processed in parallel threads, minimizing game thread work.

### 9.3 Throttled Ticks

- Each grid can define its own `UpdateFrequency`.
- Critical systems can update every frame.
- Background or backup systems can update once per second or less.

### 9.4 One-Pass Load Balancing

- Each balancing cycle uses a capped, non-recursive loop.
- Power is distributed in a single linear pass by priority group.
- Prevents infinite loops or cascading updates.

### 9.5 Grid Partitioning

- Disconnected clusters are auto-isolated and treated as separate grids.
- Balancing happens per cluster.
- No need to manually manage power regions—switches and topology define them.

---

## 10. Debugging & Visualization

Debugging tools are planned to make monitoring, testing, and tuning power flow easier.

### 10.1 Query Interfaces

- All grids and devices expose status via accessible queries:
  - Current power level
  - Priority group
  - Last update time
  - Battery status
  - Throttling state

### 10.2 Visual Overlays (Planned)

- Toggleable overlays to visualize:
  - Device status (powered, throttled, disabled)
  - Power flow direction
  - Bottlenecks and overflows
  - Network regions and IDs

### 10.3 Logging

- Optionally record:
  - Power usage per device
  - Battery activity
  - Load shedding events
  - Grid formation and topology changes

- Logs help analyze behavior over time or during edge cases.
