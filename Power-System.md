# Power System Plugin Overview

## Core Goal  
Manage a flexible, performant electricity grid system with producers, consumers, batteries (storage), switches, and hierarchical grids/subgrids—supporting priority-based load balancing, battery charge/discharge logic, event-driven updates, and dynamic grid isolation.

---

## 1. System Structure

### 1.1 World Subsystem: `UPowerSystemSubsystem`  
- Singleton managing all power grids in the world.  
- Maintains and updates all active `UPowerGrid` instances.  
- Coordinates global ticking and event dispatching.

### 1.2 Grid Classes

| Class           | Role                                                              |
|-----------------|-------------------------------------------------------------------|
| `UPowerGrid`    | Top-level controller managing multiple connected subgrids. Balances all producers and batteries globally across subgrids. Detects isolated clusters and balances them independently. |
| `UPowerSubgrid` | Logical contained power domain holding producers, consumers, and storage devices. Reports demand and status to `UPowerGrid`. Participates in global balancing or isolated local balancing if disconnected. |

---

## 2. Device Components

Each device type has a dedicated component responsible for power logic and registration:

| Component                  | Description                                                     |
|----------------------------|-----------------------------------------------------------------|
| `UPowerProducerComponent`  | Supplies power with max output, priority, and dynamic output control. Registers to its `UPowerSubgrid`. |
| `UPowerConsumerComponent`  | Requests power with load and priority. Can be load-shed if needed. Registered to `UPowerSubgrid`. |
| `UPowerStorageComponent`   | Manages stored energy (batteries, capacitors), charge/discharge rates, capacity, and priority. Registered to `UPowerSubgrid`. |
| `UPowerSwitchComponent`    | Controls flow of power on/off within or between grids. Acts as breakers or isolators. |
| `UPowerRelayComponent`     | Passive component acting as a wire or distribution box. Devices connected through it belong to the same subgrid. |

- Components register/unregister with their owning `UPowerSubgrid` during begin/end play.

---

## 3. Interfaces

Define shared interfaces to unify device and grid interactions:

| Interface                  | Purpose                                                        |
|----------------------------|----------------------------------------------------------------|
| `IPowerGridNode`           | Base interface for connectable units (grids, subgrids).        |
| `IPowerDevice`             | Common interface for producers, consumers, and storage devices.|
| `IPowerControllable`       | Interface for logic-controlled devices such as switches.       |
| `IPowerStatusReportable`   | Provides status information for debugging or UI display.       |

---

## 4. Priority & Load Balancing

- `UPowerGrid` performs **global priority-based balancing** across all connected subgrids:  
  - Producers are sorted by priority and dispatched up to max output.  
  - Batteries charge only from surplus power and discharge to meet demand respecting capacity and priority.  
- Subgrids report their aggregated demand and status upwards;  
- `UPowerGrid` allocates production and battery usage, then issues power budgets back to subgrids.  
- If subgrids become isolated (due to switch-off or breakage), they balance power independently within their cluster using the same global algorithm but scoped locally.

---

## 5. Connection & Topology

| Field            | Type            | Description                                     |
|------------------|-----------------|------------------------------------------------|
| `ActorA`         | `AActor*`       | First connected actor with a power component   |
| `ActorB`         | `AActor*`       | Second connected actor with a power component  |
| `SlotA`          | `FName`         | Connection slot name on `ActorA`                |
| `SlotB`          | `FName`         | Connection slot name on `ActorB`                |
| `ConnectionObject` | `UObject*`     | Optional visual representation (e.g., spline, cable actor) |

### Connection Rules

- Devices connected via `UPowerRelayComponent` belong to the **same subgrid**.  
- Devices connected via `UPowerSwitchComponent` are **not in the same subgrid** (switch isolates).  
- Connections represent physical or logical wiring between devices.

---

## 6. Event-Driven Updates

- Devices cache last known state (output, load, charge).  
- On any change (load, capacity, device added/removed), **dirty events** are fired.  
- Affected subgrids and grids are marked dirty and updated next tick or scheduled interval.  
- Stable parts of the grid skip updates for performance.  
- Supports forced full updates or fallback periodic syncing.

---

## 7. Additional Gameplay Features

- Subgrids can be enabled or disabled for maintenance or load shedding.  
- Switches act as circuit breakers or fuse boxes with integrated logic for scripted/automated control.  
- Relay components enable grouping without limiting power flow.  
- Supports complex priority schemes for producers and storage devices (e.g., slow-charging capacitors).  
- Visual wire layouts supported via `ConnectionObject`.

---

## 8. Communication Flow

- **Upward:** Subgrids report demand, supply, battery status, and priorities to the main grid.  
- **Downward:** The main grid issues production targets and power budgets per subgrid.  
- **Subgrid:** Distributes power to consumers based on allocated budgets and priorities.  
- **Dirty event propagation:** From devices → subgrid → grid → subsystem.

---

## 9. Performance & Scalability

- Event-driven updates minimize CPU load.  
- One-pass capped load balancing algorithm optimizes power distribution.  
- Modular grid and subgrid data allow scalability to thousands of devices.  
- Connection and switch topology management provides flexible network configuration.

---

## 10. Debug & Visualization

- Queryable interface for power flow, device status, and battery levels.  
- Planned visual overlays for load, production, and bottlenecks.  
- Logging for power usage and grid events to aid debugging and analytics.
