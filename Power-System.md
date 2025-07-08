# Power System Plugin Overview

## Core Goal  
A scalable, event-driven electricity grid system for Unreal Engine. Supports producers, consumers, storage, switches, and hierarchical subgrids—featuring priority-based global balancing, flexible grouping, visual wiring, and dynamic isolation handling.

---

## 1. System Structure

### 1.1 World Subsystem: `UPowerSystemSubsystem`  
- Singleton managing all power grids in the world.  
- Owns and updates all active `UPowerGrid` instances.  
- Coordinates global ticking and event propagation.

### 1.2 Grid Classes

| Class           | Role                                                              |
|-----------------|-------------------------------------------------------------------|
| `UPowerGrid`    | Central controller. Balances all producers and batteries globally across all connected `UPowerSubgrid`s. Detects and handles isolated groups. |
| `UPowerSubgrid` | Logical power domain containing producers, consumers, and storage. Reports demand and participates in global balancing. |

---

## 2. Device Components

Each device type is modularized with components that register with their subgrid:

| Component                  | Description                                                       |
|----------------------------|-------------------------------------------------------------------|
| `UPowerProducerComponent`  | Supplies power. Has max output, dynamic capacity, and priority.   |
| `UPowerConsumerComponent`  | Draws power. Has load, priority, and can be load-shed.            |
| `UPowerStorageComponent`   | Batteries or capacitors. Manages charge/discharge with priority.  |
| `UPowerSwitchComponent`    | Logical breaker isolating devices or subgrids.                    |
| `UPowerRelayComponent`     | Passive connection grouping devices into the same subgrid.       |

---

## 3. Interfaces

| Interface                  | Description                                               |
|----------------------------|-----------------------------------------------------------|
| `IPowerGridNode`           | Generic interface for connectable grids and subgrids.    |
| `IPowerDevice`             | Shared interface for producers, consumers, and storage.  |
| `IPowerStatusReportable`   | For debugging and UI status queries.                      |
| `IPowerControllable`       | For togglable devices like switches.                      |

---

## 4. Load Balancing and Priorities

- `UPowerGrid` globally balances all producers and batteries across connected subgrids.  
- Producers sorted by priority and capped at max output.  
- Batteries charge from surplus power and discharge during deficits, respecting capacity and priority.  
- `UPowerSubgrid` reports total demand and device registrations; receives power budget to distribute to consumers.

---

## 5. Connection & Topology

| Field            | Type            | Description                                     |
|------------------|-----------------|------------------------------------------------|
| `ActorA`         | `AActor*`       | First connected actor with a power component   |
| `ActorB`         | `AActor*`       | Second connected actor with a power component  |
| `SlotA`          | `FName`         | Connection slot name on `ActorA`                |
| `SlotB`          | `FName`         | Connection slot name on `ActorB`                |
| `ConnectionObject` | `UObject*`     | Optional visual representation (e.g., spline)  |

### Connection Rules

- Devices connected via `UPowerRelayComponent` belong to the **same subgrid**.  
- Devices connected via `UPowerSwitchComponent` are **not in the same subgrid** (switch isolates).  
- Connections define the physical or logical wiring between devices.

---

## 6. Event-Driven Updates

- Components cache last known states.  
- State changes mark owning subgrid or grid as **dirty** for update.  
- Dirty grids update on next tick or scheduled cycle.  
- Full sync can be forced or run periodically.

---

## 7. Gameplay Features

- Subgrids can be toggled on/off via switches for load shedding or maintenance.  
- Switches isolate subgrids or device groups.  
- Relay components group devices without limiting power flow.  
- Supports complex priority schemes for producers and storage devices.  
- Visual wire layouts supported through `ConnectionObject`.

---

## 8. Communication Flow

1. Devices update state and mark subgrids dirty.  
2. Subgrids report demand and device status to the main grid.  
3. `UPowerGrid` computes global power flow respecting priorities.  
4. Power budgets are sent back to subgrids for consumer distribution.  
5. Isolated subgrids perform balancing independently within their partitions.

---

## 9. Performance & Scalability

- Event-driven updates minimize CPU usage.  
- Centralized load balancing avoids redundant calculations.  
- Supports large, modular power networks with thousands of devices.  
- Topology and connection management allows clear and flexible wiring.

---

## 10. Debug & Monitoring

- Planned overlays visualize load, power flow, and bottlenecks.  
- Query interface for device and grid status.  
- Connection objects enable visual debugging of wiring.
