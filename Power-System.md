# Power System Overview

## Core Goal  
Manage a scalable, event-driven electricity grid system for Unreal Engine. The system supports producers, consumers, storage, and hierarchical subgrids—balancing power with global priority logic and dynamic isolation handling.

---

## 1. System Structure

### 1.1 World Subsystem: `UPowerSystemSubsystem`  
- Singleton managing all power grids in the world.  
- Owns and updates all active `UPowerGrid` instances.  
- Coordinates global ticking and change propagation.

### 1.2 Grid Classes

| Class           | Role                                                              |
|-----------------|-------------------------------------------------------------------|
| `UPowerGrid`    | Central controller. Balances all producers and batteries globally across all connected `UPowerSubgrid`s. Detects and handles isolated groups. |
| `UPowerSubgrid` | Represents a physical or logical power group. Contains **producers, consumers, and storage**. Participates in global balancing or operates locally if isolated. |

---

## 2. Device Components

Each device type has a dedicated component for modular control and integration:

| Component                  | Description                                                    |
|----------------------------|----------------------------------------------------------------|
| `UPowerProducerComponent`  | Supplies power to the subgrid. Has max output, dynamic capacity, and priority level. |
| `UPowerConsumerComponent`  | Draws power from the subgrid. Has priority, target consumption, and can be load-shed. |
| `UPowerStorageComponent`   | Represents batteries or capacitors. Can charge/discharge. Obeys charge rates, capacity, and global priority. |
| `UPowerSwitchComponent`    | Logic-controlled on/off switch for devices or subgrid isolation. Optional. |

- Components auto-register with their parent `UPowerSubgrid` during `BeginPlay`.

---

## 3. Interfaces

| Interface                  | Description                                               |
|----------------------------|-----------------------------------------------------------|
| `IPowerGridNode`           | Generic interface for connectable grids/subgrids.         |
| `IPowerDevice`             | Implemented by producers, consumers, and storage.         |
| `IPowerStatusReportable`   | Used for debugging/UI to fetch device state.              |
| `IPowerControllable`       | Allows toggling or logic-based control of power devices.  |

---

## 4. Load Balancing and Priorities

### Global Balancing
- When all subgrids are connected, `UPowerGrid`:
  - Balances all **producers and batteries** globally.
  - Distributes load to **highest priority producers** first.
  - Charges batteries only with surplus, respecting charge rate and priority.

### Subgrid Participation
- Each `UPowerSubgrid`:
  - Reports total demand.
  - Registers all its devices (producers, consumers, storage).
  - Receives and applies its power budget.

### Isolation Behavior
- If subgrids become **disconnected or toggled off**, they:
  - Form independent **power clusters**.
  - Perform **local balancing** internally within the isolated group.
  - Still obey producer/storage priorities, just scoped locally.

---

## 5. Event-Driven Updates

- Each component tracks its last known state.  
- Any change (load, output, charge level`-Only on fully charged or discharged`) marks the owning subgrid/grid as **dirty**.  
- Dirty subgrids are re-evaluated during the next tick/update cycle.  
- Fully event-driven; avoids per-frame updates unless state changes.  
- Fallback: periodic full sync.

---

## 6. Gameplay Features

- **Subgrid control:** Subgrids can be enabled or disabled for maintenance, load shedding, or simulation purposes.  
- **Switch components:** Allow logic-driven toggling of devices or entire subgrids.  
- **Storage prioritization:** Slow-charging backup batteries can be deprioritized.  
- **Future extensibility:** Logistics systems can take over high-level scripting.

---

## 7. Communication Flow

1. **Subgrid reports total demand and device info** to `UPowerGrid`.  
2. **`UPowerGrid` computes power flow**, considering all producers, consumers, and storage.  
3. **Power budgets and targets are sent** back to each subgrid.  
4. **Subgrid applies logic locally**, distributing power to its devices.  
5. If isolated, the subgrid handles **balancing independently** using the same algorithm.

---

## 8. Performance & Scalability

- Designed for large worlds with thousands of devices.  
- Centralized balancing avoids redundant logic.  
- Dirty flag system avoids unnecessary ticking.  
- Isolation handling supports fully modular grid expansions.

---

## 9. Debug & Monitoring

- Visual overlays (planned): show power flow, loads, and bottlenecks.  
- Reporting interface: query production, consumption, and battery levels.  
- Logging of power events and system changes for debugging or analytics.

