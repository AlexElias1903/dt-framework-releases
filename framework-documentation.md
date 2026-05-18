# DT Framework Documentation

> **Version:** 2.9.6
> **Audience:** researchers, developers, and integrators building Digital Twin scenarios
> **Status:** working draft

The DT Framework is a domain-agnostic platform for modeling, simulating, and observing IoT-driven Digital Twin scenarios. The same primitives apply to logistics, healthcare, smart cities, manufacturing, energy, agriculture, or any other domain where you have a set of entities, devices producing data, and rules that react to that data. This document describes every concept, tool, and integration point exposed by the framework so you can adapt them to your own scenario.

Concrete examples in this document use a small warehouse scenario (containers, a forklift, an RFID scanner) to illustrate ideas. They are illustrative only; the framework itself contains no domain logic.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Core Concepts](#3-core-concepts)
4. [The Desktop Launcher](#4-the-desktop-launcher)
5. [Working with DT Types](#5-working-with-dt-types)
6. [Modeling Entities (Instance Types and Instances)](#6-modeling-entities-instance-types-and-instances)
7. [Wiring Devices and Messages](#7-wiring-devices-and-messages)
8. [Writing Algorithms](#8-writing-algorithms)
9. [Visualization Tools](#9-visualization-tools)
10. [Observability: Live Feed, Alerts, Commands, Telemetry](#10-observability-live-feed-alerts-commands-telemetry)
11. [Persistence and File Layout](#11-persistence-and-file-layout)
12. [Distribution: `.dtpkg` Packages](#12-distribution-dtpkg-packages)
13. [Algorithm Context API Reference](#13-algorithm-context-api-reference)
14. [Reproducibility: Baselines and Resets](#14-reproducibility-baselines-and-resets)
15. [REST API and External Integration](#15-rest-api-and-external-integration)
16. [Glossary](#16-glossary)

---

## 1. Overview

The DT Framework lets you build a Digital Twin (DT) for any domain by configuring everything through a desktop application. Instead of writing a bespoke simulator from scratch, you describe your domain in terms of generic primitives — entities, messages, devices, algorithms — and the platform handles persistence, message routing, scheduling, and visualization.

A scenario in the DT Framework is a graph of entities producing and consuming messages. The platform records every message, persists every state change, and gives you a 2D map view, time-series telemetry, and an alerting system out of the box.

The framework is delivered as an Electron desktop application (the *launcher*) that bundles a full Docker stack: PostgreSQL, RabbitMQ, MQTT (Mosquitto), Redis, a Python backend (FastAPI), a React frontend, and per-DT-Type *DT containers* that execute user-defined algorithms in isolation.

---

## 2. Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                  Desktop Launcher (Electron)                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  React UI (frontend, served on http://localhost:3001)    │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                       Backend (FastAPI)                        │
│  REST API · DB access · Container orchestration · Auth         │
└────────────────────────────────────────────────────────────────┘
            │              │              │
            ▼              ▼              ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │  PostgreSQL  │  │   RabbitMQ   │  │   Mosquitto  │
   │   (state)    │  │  (commands)  │  │    (MQTT)    │
   └──────────────┘  └──────────────┘  └──────────────┘
                              │
                              ▼
                  ┌────────────────────────┐
                  │   DT Containers (1 per │
                  │   DT Type, isolated)   │
                  │   running algorithms   │
                  └────────────────────────┘
                              │
                              ▼
                       Bridge + Enricher
                       (telemetry pipeline)
```

### Services

| Service | Role |
| --- | --- |
| **Launcher** (Electron) | Manages the lifecycle of the Docker stack, exposes a desktop UI for service status, logs, updates, and configuration. |
| **Frontend** (React) | The main interaction surface. Every modeling and inspection action happens here. |
| **Backend** (FastAPI) | Owns the database, exposes the REST API, orchestrates the per-DT-Type containers. |
| **PostgreSQL** | Persistent state: DT Types, Instance Types, Instances, Algorithms, Messages, Devices, Alerts, Telemetry. |
| **RabbitMQ** | Internal command/event bus between services. |
| **Mosquitto (MQTT)** | Ingress for messages from external physical devices. |
| **Redis** | Caching layer and short-term state. |
| **DT Container** (per DT Type) | Sandboxed Python runtime that executes the user-defined algorithms for one scenario. Failures in one container do not affect others. |
| **Bridge** | Subscribes to MQTT, routes incoming messages to the right DT container. |
| **Enricher** | Decorates raw telemetry with metadata (timestamps, device names, hierarchical paths) before storing. |

### Data Flow

1. A device (synthetic or physical) emits a message.
2. The message lands in the DT container of its DT Type via Bridge or via the synthetic scheduler.
3. Generation algorithms produce messages on a schedule or on demand.
4. Processing algorithms react to incoming messages and may emit further events, mutate Instance state, or raise alerts.
5. The Enricher records every message and state change to PostgreSQL.
6. The frontend polls the backend at 1 Hz to refresh the Space View, Live Feed, and Alerts panel.

---

## 3. Core Concepts

The framework exposes seven primitives. Mastering these is enough to model almost any scenario.

### 3.1 Digital Twin Type (DT Type)

A **DT Type** is the top-level container for a scenario. It groups everything that belongs to one logical project. Creating a DT Type provisions a dedicated Docker container that will run that scenario's algorithms and isolate its state from other DT Types.

Each DT Type has:
- A unique `type_name` (used as the container identifier).
- Lifecycle status: `inactive`, `active`, `running`, `stopped`, `error`.
- A self-contained set of Instance Types, Instances, Algorithms, Message Definitions, Devices, and floorplans that live inside it.

Typically you create one DT Type per scenario you want to study. The DT Type is the unit of deployment, export (`.dtpkg`), and reset.

### 3.2 Instance Type

An **Instance Type** is the template for a kind of entity in your domain. Examples: a "moving asset" template, a "static area" template, a "sensor" template. Think of it as a class that defines:

- **Category**: `environment` (drawn as a footprint outline, used for areas/zones that frame other entities), `passive` (rendered as icons/shapes; entities whose physical position is typically stable, though their `consolidated_state` can still evolve), or `active` (rendered as icons/shapes; entities whose behavior is the focus — typically moving, but more broadly any entity that should be visually highlighted in stacks). The category affects rendering, stacking, and default layer order; it does **not** restrict what algorithms can do to the instance — any category can have its state and position updated by algorithms.
- **Visual properties** (Visual 2D panel):
  - Built-in icon (`circle`, `square`, `diamond`, `truck`, `rectangle`) or uploaded PNG/SVG.
  - Default color (hex string).
  - Marker scale (legacy multiplier when `physical_dimensions` is unset).
  - Layer order (z-index controlling render and hit-test order in the 2D viewer).
  - Opacity (0.0–1.0).
- **Default dimensions** (`physical_width`, `physical_height`, `physical_depth` in meters): inherited by Instances of this type unless overridden.
- **Static flag**: hint that instances of this type do not move; influences default layer ordering when no explicit layer order is set.
- **Color rules** (optional, [section 6.4](#64-color-rules)): conditional color changes based on `consolidated_state` fields.
- **Device slots** (optional, [section 6.5](#65-device-slots)): named slots that Instances of this type expect to bind to specific Device categories.

### 3.3 Instance

An **Instance** is a concrete entity, an "object" of a given Instance Type. It has:

- **Instance ID**: a stable string identifier you choose (lowercase with hyphens recommended).
- **Instance Name**: human-readable label.
- **Manual position** (`x`, `y`, `z` in meters): location on the 2D map.
- **Physical dimensions** (`width`, `height`, `depth` in meters): footprint, overrides the type default.
- **Parent ID**: optional reference to another Instance, forming a hierarchy.
- **Consolidated state** (free-form JSON): the live state of the instance. Algorithms read and patch this field to drive behavior.
- **Baseline state**: a snapshot of `manual_position`, `consolidated_state`, and `parent_id` captured at creation and re-savable on demand. Powers the Reset functionality.
- **Virtual flag**: `true` for synthetic/simulated entities, `false` for entities backed by real-world sensors.
- **Status**: lifecycle marker (`active`, `inactive`).

Instances are the unit of state in the framework. Algorithms typically iterate over Instances, mutate `consolidated_state`, move them via `update_instance_position`, or change their hierarchy via `update_instance_parent`.

### 3.4 Device Type

A **Device Type** describes a kind of device that emits or consumes messages. Each Device Type is associated with one Message Definition: the schema of the messages it traffics.

Device Types are scoped to a DT Type so the same name can coexist across scenarios.

### 3.5 Device (synthetic and physical)

A **Device** is a concrete instance of a Device Type. There are two kinds:

- **Physical**: the platform expects messages to arrive from outside (typically over MQTT). Bridge consumes the configured topic and routes messages into the DT container.
- **Synthetic**: the device is simulated by an algorithm running in the DT container. The framework treats synthetic devices identically to physical ones in terms of message routing, state, and visualization.

Each Device has a unique identifier and may be associated with an Instance via the device-slot mechanism. Devices have their own state fields:
- `device_context` (free-form JSON, intended for per-device configuration)
- `position` and `floorplan_position`
- `color` and `geometry` (visual hints)
- `parent_id` (hierarchy of devices, used for drill-down floorplans)

### 3.6 Message Definition

A **Message Definition** describes a JSON message format used to communicate inside the DT Type. It has a name and a schema.

Schemas are **inferred from a JSON example**. You paste a representative payload and the platform derives field names and types automatically. There is no need to hand-write JSON Schema, Avro, or protobuf definitions.

For example, you paste this payload as the example for a message definition named `SENSOR_READING`:

```json
{
  "sensor_id": "temp-01",
  "value": 23.5,
  "unit": "celsius",
  "timestamp": "2026-05-04T18:30:00Z",
  "alerts": []
}
```

The platform infers a schema with five fields: `sensor_id` (string), `value` (number), `unit` (string), `timestamp` (string), `alerts` (array). All future messages of this type are validated against that shape.

Messages are the only way algorithms communicate. An algorithm produces messages of a given type, another algorithm processes them, and the platform stores every message for replay and analytics.

### 3.7 Algorithm

An **Algorithm** is a Python function that the DT container executes. There are two kinds:

- **Generation algorithm** (`algorithm_type='generation'`): produces messages.
  - **Periodic** (`trigger_type='periodic'`): runs every `generation_interval` seconds.
  - **Manual** (`trigger_type='manual'`): runs only when explicitly triggered via the **Run now** button or the API. Optional `params` JSON is passed to the algorithm.
- **Processing algorithm** (`algorithm_type='processing'`, `trigger_type='on_data'`): runs in response to a specific message arriving.

Each algorithm exposes one entry point:

- Generation algorithms implement `def generate_data(context):` and return a dict (the message payload), `None` to skip this tick, or a list of dicts to emit several messages.
- Processing algorithms implement `def process_data(data, context):` where `data` is the incoming message payload. They typically return `data` (possibly enriched).

The `context` argument exposes the algorithm API documented in [section 13](#13-algorithm-context-api-reference).

Algorithms are persisted as Python source code. The platform compiles and runs them inside the per-DT-Type container, so a syntax error in one DT Type cannot break others.

---

## 4. The Desktop Launcher

The desktop application is the main entry point for end users. After installing the platform-specific package (`.dmg`, `.exe`, `.AppImage`, `.deb`), launching the app shows a sidebar with five sections.

### Home

Entry point. Surfaces the global state of the stack: number of running services, last log line, quick **Start / Stop** buttons.

### Services

Lists every Docker service the framework manages. For each service you see image tag, container status, port mappings, last health check, per-service Start/Stop/Restart actions, and a button to open the matching logs view.

### Logs

A scrollable, filterable view of every container's stdout/stderr. Useful when debugging an algorithm that fails to start or a backend error.

### Updates

Polls the public release feed and shows the latest available version. Two update channels:

- **Electron app**: replaces the installer itself when a newer release is available. Auto-update follows the `latest*.yml` files published with each release.
- **Docker images**: pulls newer image tags for backend, frontend, bridge, enricher when the configured `dt-config.yaml` advertises a higher version. Allows updating individual services without reinstalling the desktop app.

### Config

Edits the runtime `dt-config.yaml` (image tags, ports, paths, log retention). Changes regenerate the Docker Compose file and prompt for a restart of the affected services.

---

## 5. Working with DT Types

Open the **Digital Twin Types** tab in the frontend (port 3001) to manage scenarios.

### Creating a DT Type

1. Click **+ New Digital Twin Type**.
2. Pick a name (lowercase with underscores recommended).
3. Optionally provide a description and a version string (free-form, e.g. `1.0.0` — useful for tagging exports of an evolving scenario).
4. The platform provisions a dedicated Docker container with the Python runtime and helper modules.
5. Status moves to `active` once the container starts.

### DT Type Detail Tabs

Each DT Type has its own detail page with these tabs:

- **Message Definitions**: register the JSON schemas used in this scenario.
- **Algorithms**: write generation and processing algorithms.
- **Devices**: configure synthetic or physical devices and their device types.
- **DT Instances**: manage entities and their state, plus the Instance Types catalog.
- **Live Feed**: real-time telemetry stream of messages produced inside this DT Type.
- **Dashboard**: build and view custom dashboards (charts, KPIs, tables) over the message history.
- **Commands**: send control commands to specific devices or instances (manual interventions).
- **Space View**: 2D map showing the spatial layout of all instances.
- **Advanced**: container management, Space View configuration, per-device floorplan editor.

### MQTT Instructions

The Message Definitions tab includes a panel showing the MQTT topics and credentials that physical devices should use to publish into this DT Type. Useful when integrating real hardware.

### Lifecycle Operations

See [section 14](#14-reproducibility-baselines-and-resets) for the details of baseline snapshots, reset operations, and reset experiment behavior.

### Export / Import

A DT Type can be exported as a `.dtpkg` archive (see [section 12](#12-distribution-dtpkg-packages)) that bundles its Instance Types, Instances, Algorithms, Message Definitions, Devices, uploaded icons, and floorplans. Importing a `.dtpkg` recreates the entire scenario in another framework instance.

---

## 6. Modeling Entities (Instance Types and Instances)

### 6.1 Creating an Instance Type

1. In the DT Instances tab, scroll to **Instance Types** and click **+ New Type**.
2. Fill in:
   - **Type Name**: the class name for this kind of entity.
   - **Category**: Active, Passive, or Environment. See [section 3.2](#32-instance-type) for the distinction. The category is mainly a rendering hint — algorithms can mutate state on any category.
   - **Description** (optional).
   - **Default Color**: hex string used as the marker fill.
3. Expand **Visual 2D**:
   - Choose a **Built-in Icon** OR upload a custom image (PNG/SVG/JPEG/WebP).
   - Set **Marker Scale** (size multiplier when `physical_dimensions` is unset).
   - Set **Layer Order** (z-index): lower values render and hit-test behind higher values. Use any monotonically increasing sequence.
4. Expand **Default Dimensions**:
   - **Width**, **Height**, **Depth** in meters.
   - **Opacity** (0.0–1.0).
5. Optionally configure **Color Rules** ([section 6.4](#64-color-rules)) and **Device Slots** ([section 6.5](#65-device-slots)).
6. Save.

### 6.2 Creating an Instance

1. In the DT Instances tab, click **+ New Instance**.
2. Pick the **Instance Type** from the dropdown. The form auto-fills the default dimensions from the type.
3. Fill in:
   - **Instance ID**: stable identifier, lowercase with hyphens.
   - **Friendly Name**: any human-readable label.
   - **Parent Instance**: optional, builds the hierarchy.
   - **Physical / Virtual**: pick **Virtual** for synthetic, **Physical** when backed by real hardware.
   - **State (consolidated_state)**: paste a JSON object representing the initial state. Optional; defaults to `{}`.
4. Position the instance on the **mini-map** (the floorplan preview embedded in the modal):
   - **Click** anywhere to set X/Y as a point.
   - **Click and drag** to draw a rectangle that sets X/Y (center) AND the physical dimensions (width × depth).
   - The Z field controls vertical stacking.
5. Save. The platform also captures this state as the instance's baseline.

### 6.3 Hierarchy and Stacking

- **Hierarchy** is logical: `parent_id` references another Instance. Algorithms can navigate via the parent reference.
- **Stacking** is physical: same X/Y, different Z. The Space View groups stacked items into a single marker with a count badge; clicking opens the Stack Panel listing each item top-down.

### 6.4 Color Rules

Color Rules let an Instance Type change its marker color based on a field value in `consolidated_state`. Each rule is a tuple `(field, value, color)`. At render time the framework checks the rules in order and applies the first match; otherwise the default color is used.

Example: a sensor template with a `status` field can show different colors per state:

```json
[
  { "field": "status", "value": "ok", "color": "#52c41a" },
  { "field": "status", "value": "warning", "color": "#faad14" },
  { "field": "status", "value": "error", "color": "#ff4d4f" }
]
```

If an Instance has `consolidated_state.status = "warning"`, its marker is rendered in yellow. If the field is missing or has any other value, the template's default color is used.

Use cases:
- Highlight an entity in error state.
- Show a sensor's reading band (green/yellow/red zones).
- Distinguish loaded vs empty mobile assets.

### 6.5 Device Slots

Device Slots formalize the relationship between Instances and Devices. An Instance Type declares one or more named slots, each constrained to a `device_type_id` (the device-catalog entry) and marked as required or optional. When you create an Instance and attach Devices to it, the framework validates the slots are filled.

Example: a "tracked asset" template might declare three slots, each pointing at a Device Type from the catalog:

```json
[
  { "slot_name": "gps",    "device_type_id": "<uuid of gps_sensor>", "required": true },
  { "slot_name": "tag",    "device_type_id": "<uuid of rfid_tag>",   "required": true },
  { "slot_name": "camera", "device_type_id": "<uuid of camera>",     "required": false }
]
```

In the UI you do not see the UUID — the slot editor shows the Device Type name from the catalog (`gps_sensor`, `rfid_tag`, `camera`) and stores the corresponding UUID under the hood.

Every Instance of this type expects to have a GPS device and an RFID tag bound to it; a camera is optional. The framework rejects Instance creation if a required slot is unbound.

Use cases:
- Assets that require both a position sensor and an identifier.
- Monitoring stations with optional secondary devices.
- Vehicles with slots for cargo and driver identification.

### 6.6 Editing Positions Visually in the Space View

The Space View has an **Edit positions** toggle in the top-left corner. When enabled:
- Every marker becomes draggable.
- Releasing the marker issues a `PATCH /instances/{id}/position` automatically.
- Useful for fine-tuning positions visually after creation.

The Edit positions toggle changes only the X/Y of `manual_position`. Z is preserved.

---

## 7. Wiring Devices and Messages

### 7.1 Defining Messages

1. Open **Message Definitions**.
2. Click **+ New**.
3. Pick a name.
4. Paste a JSON example representing one message of this type.
5. The framework infers the schema and stores it.

### 7.2 Defining Device Types

1. Open the **Device Types** sub-tab inside Devices.
2. Click **+ New Device Type**.
3. Pick a name and the Message Definition it produces or consumes.

### 7.3 Creating a Synthetic Device

1. Open **Devices → Create**.
2. Fill in identifier, name, type, and pick **Synthetic** mode.
3. Save. The device is now eligible to be the source of generation algorithms or the consumer of processing algorithms.

### 7.4 Creating a Physical Device

Same flow, but pick **Physical** mode. The framework expects messages to arrive on a configured MQTT topic and routes them to the DT container automatically. The Message Definitions tab shows the exact topic and credentials.

### 7.5 Data Flows: Binding Algorithms to Devices

Creating an algorithm and creating a device are not enough to make data start flowing. You have to **bind** them on the device via a **Data Flow**. A Data Flow declares: "for *this* device, run *this* generation algorithm (or accept *this* incoming Message Definition), and pipe the result through *these* processing algorithms."

Without a Data Flow nothing fires: the algorithm exists in the catalog and the device exists in the model, but they never meet.

To create a Data Flow:

1. Open **Devices** in the DT Type detail and click the device you want to wire (synthetic or physical).
2. Open the **Data Flows** sub-tab on the device detail page.
3. Click **+ New Data Flow** and fill in:

| Field | Purpose |
| --- | --- |
| **Flow Name** | Human-readable label, must be unique on the device. |
| **Flow Type** | Auto-derived from the device: `synthetic` for synthetic devices, `physical` for physical ones. |
| **Generation Algorithm** *(synthetic flows only)* | The algorithm whose output drives the flow. Required for synthetic flows. |
| **Message Definition** *(physical flows only)* | The Message Definition expected from the external MQTT feed. Required for physical flows. |
| **Processing Algorithms (pipeline)** | Zero or more processing algorithms applied in order to every message produced by the flow. Each algorithm's return value is the input to the next. |
| **Description** | Optional free text. |
| **Status** | `Active` schedules the flow and routes incoming messages; `Inactive` disables it without deleting it. |

Behavior:

- A **periodic generation algorithm** ticks every `generation_interval` seconds *only* while it is referenced by an active synthetic Data Flow.
- A **manual generation algorithm** is fired by the **Run now** button on the algorithm itself, but the messages it produces still travel through the bindings declared on the Data Flow that references it (so the processing pipeline runs).
- For **physical devices**, the flow tells the Bridge which processing pipeline to apply to each MQTT message of the configured Message Definition.

A device can have multiple Data Flows (e.g. a synthetic flow for simulation and a physical flow for real hardware, switched by toggling Active), and the same algorithm can be referenced by Data Flows on multiple devices — each binding is independent.

### 7.6 Per-device Floorplans (Drill-down)

A device can have its own floorplan that appears when the user clicks the device in the Space View. This drill-down is recursive: each floorplan can contain children with their own floorplans.

Use cases:
- Click a building marker to enter its floor layout.
- Click a shelf marker to see individual slots.
- Click a vehicle to see internal compartments.

The Floorplan Editor is in the **Advanced** tab of the DT Type detail page.

---

## 8. Writing Algorithms

The **Algorithms** tab is where the scenario logic lives.

### 8.1 Algorithm Structure

Every algorithm is a Python script with a single entry point.

**Generation algorithm:**

```python
def generate_data(context):
    # Read state, compute, return a payload (or None to skip).
    return {"key": "value"}
```

**Processing algorithm:**

```python
def process_data(data, context):
    # `data` is the incoming message payload.
    # Mutate state, raise alerts, return data (possibly modified).
    return data
```

### 8.2 Trigger Modes

| Mode | Behavior | Typical use |
| --- | --- | --- |
| `trigger_type='periodic'` (generation) | Runs every `generation_interval` seconds while bound to an active synthetic Data Flow ([section 7.5](#75-data-flows-binding-algorithms-to-devices)). Without an active flow it never fires. | Sensor simulators, world-state updates, clocks, periodic sweeps. |
| `trigger_type='manual'` (generation) | Runs only when explicitly triggered via **Run now** or the API. Stays out of the periodic scheduler. The processing pipeline of any Data Flow that references it still runs. | User-initiated events, on-demand workflows. |
| `trigger_type='on_data'` (processing) | Subscribes to a Message Definition. Runs once for every incoming message of that type that travels through a Data Flow whose pipeline includes this processing algorithm. | Reactive logic, anomaly detection, derived metrics. |

### 8.3 The Two UI Buttons: "Execute" vs "Run now"

The Algorithms tab shows two action buttons next to each algorithm.

| Button | Endpoint | When to use |
| --- | --- | --- |
| **Execute** | `POST /algorithms/{id}/execute-generation` | **Test mode**. Runs the algorithm code in a sandbox for every device that matches and returns the generated payloads in a modal so you can inspect them. Does **not** emit through the live pipeline. Generation-only. Use during development to validate algorithm code returns the right shape. |
| **Run now** | `POST /algorithms/{id}/trigger` | **Real trigger**. Fires the algorithm through the live messaging pipeline: produced messages are published, downstream processors run, telemetry is recorded, alerts may fire. Generation-only. Optional `params` JSON is passed to `context["params"]`. |

Rule of thumb: **Execute** to debug, **Run now** to actually run.

For periodic algorithms, the regular scheduler keeps firing on the configured interval; **Run now** is mainly useful for `trigger_type='manual'` algorithms that have no schedule.

For processing algorithms, neither button applies in the usual way: processing algorithms are driven by incoming messages. To "test" a processing algorithm, use **Run now** on a generation algorithm that produces the message it listens to.

### 8.4 Imports and Sandboxing

Algorithms run in a constrained Python environment that exposes only the standard library. Useful modules: `datetime`, `json`, `math`, `random`, `statistics`, `urllib.request`, `uuid`, `re`, `collections`. Third-party packages (numpy, pandas, requests, etc.) are not available.

If you need an external package, do the work outside the framework and either:
- Pre-compute and store results in `consolidated_state` before triggering, or
- Use the `webhook` helper to call out to a service you control.

### 8.5 Returning Messages

A generation algorithm can return:
- `None`: skip this tick (no message produced).
- A `dict`: emits one message with this payload.
- A `list[dict]`: emits multiple messages.

A processing algorithm can return the (possibly modified) `data` dict; the modified payload is what downstream consumers will see and what gets persisted.

### 8.6 Error Handling

If an algorithm raises an exception, the platform logs it (visible in the **Logs** tab) and continues. A misbehaving algorithm cannot crash the DT container; it just stops producing for that tick. Always test algorithms with **Execute** before relying on them in **Run now** scenarios.

---

## 9. Visualization Tools

### 9.1 Space View (2D Viewer)

A pannable, zoomable 2D map of the DT Type's spatial layout.

| Action | How |
| --- | --- |
| Pan | Click and drag on empty canvas. |
| Zoom | Mouse wheel (anchored at cursor position). |
| Reset view | The **R** button on the right. |
| Edit positions | Toggle in the top-left corner. Enables drag-to-reposition for instance markers. |
| Hover | Tooltip with instance name. Honors `layer_order` for hit-testing. |
| Click marker | Opens the Marker Popup. |
| Click stack badge | Opens the Stack Panel. |

#### Render Rules

- **Environment** instances: rectangle outline drawn from `physical_dimensions` (width × depth), translucent fill.
- **Active** and **Passive** instances: image marker (`image_2d_path`) or built-in shape, sized by `physical_dimensions` if set, otherwise `MARKER_SIZE × marker_scale`.
- Markers are sorted by `layer_order` ascending: lower draws first (back), higher draws last (front).

#### Floorplan Background

The DT Type can have a root floorplan (an image of the area being modeled) configured via **Advanced → Space View Configuration**. You upload an image and provide the real-world dimensions in meters. The platform scales coordinates to fit.

#### Drill-down

Each Device can have its own floorplan. Clicking a Device marker that has children navigates into its sub-floorplan, showing the device's children as new markers. A breadcrumb at the top lets you navigate back.

### 9.2 Marker Popup

Clicking an Instance marker opens a side popup showing:
- Instance name and type (with the icon if uploaded).
- Full `consolidated_state` as a formatted key/value list.
- Position and dimensions.
- Linked devices (with quick links to telemetry).
- Action buttons: Edit, Delete, Save baseline, Reset.

### 9.3 Stack Panel

When multiple Instances share the same X/Y but differ in Z, they collapse into a single marker with a count badge. Clicking the badge opens the Stack Panel showing every item in the stack sorted by Z (top first), with TOP/BOTTOM markers and a per-row click that opens its individual Marker Popup.

### 9.4 Dashboard Manager

Build custom dashboards from telemetry. Create charts (line, bar, scatter), KPIs, and tables driven by message history. Useful for exporting figures into a paper or thesis without leaving the framework.

### 9.5 Telemetry Viewer

Per-device historical view that plots numeric fields over time. Reachable from the Marker Popup via the **Telemetry** button. Ideal for inspecting a specific sensor's evolution without building a full dashboard.

---

## 10. Observability: Live Feed, Alerts, Commands, Telemetry

### 10.1 Live Feed

A real-time stream of every message in the DT Type. Each row shows timestamp, message type, source device, and a summary of the payload. Click any row to expand and see the full JSON payload with proper indentation. Useful for debugging algorithms and verifying the data flow.

#### History scrubber and density heatmap

Above the feed there is a time scrubber with a heatmap that lets you inspect history without writing a database query:

- **Heatmap colors**: dark green = many events in that bucket; light green = moderate; gray dashed = no data (a real gap, not a UI artifact).
- **Cursor**: drag the cursor onto a bucket to load the 60-second window of events ending at that moment. The list below updates accordingly.
- **Live mode**: when the cursor is at the right edge, the feed streams new messages as they arrive. Drag back to inspect history; click **Live** to snap back to the present.

This is the primary place to re-investigate past telemetry without re-running the scenario.

### 10.2 Alerts

The global **Alerts** tab (top of the page, outside any DT Type) collects alerts emitted by `context["alert"](severity, message)` from processing algorithms. Each alert has:
- Severity (`info`, `warning`, `critical`).
- Message string.
- DT Type and source device.
- Timestamp.
- Read / unread state.

Alerts are append-only in the current version: there is no "resolved" state. Algorithms can emit a follow-up `info` alert as a workaround when a condition is corrected.

### 10.3 Commands

The **Commands** tab lets you send a command message to a specific device or instance. Useful for manual interventions during testing without writing a separate UI.

### 10.4 Container Status

In the DT Type **Advanced** tab, the Container Status panel exposes the per-DT-Type Docker container's lifecycle (running, stopped, errored), recent log tail, and Start/Stop/Restart/Rebuild controls. Useful when an algorithm change requires rebuilding the container.

---

## 11. Persistence and File Layout

The framework persists everything to disk through Docker bind mounts. The host paths depend on the OS where the desktop launcher runs.

### Per-OS Storage Root

The launcher uses Electron's `app.getPath('userData')` joined with `runtime/`:

| OS      | Path                                                            |
| ------- | --------------------------------------------------------------- |
| macOS   | `~/Library/Application Support/dt-framework/runtime/`           |
| Windows | `%APPDATA%\dt-framework\runtime\`                               |
| Linux   | `~/.config/dt-framework/runtime/`                               |

### Layout Inside `runtime/`

```
runtime/
├── dt-config.yaml                  # User-edited runtime config
├── docker-compose.generated.yml    # Auto-generated Compose file
├── data/
│   ├── postgres/                   # PostgreSQL data dir (state)
│   ├── uploads/                    # User-uploaded images and floorplans
│   │   ├── instance-type-icons/    # Per-template PNG/SVG
│   │   ├── floorplans/             # Per-DT-Type floorplans
│   │   └── _dtpkg_staging/         # Temporary staging for .dtpkg ops
│   └── generated_containers/       # Per-DT-Type Python source generated by the backend
├── logs/                           # Per-service log files
└── backups/                        # DT Type backups (when explicitly created)
```

### Backups

To back up a user's full state, copy the `runtime/` directory entirely. To share a single scenario, prefer `.dtpkg` export.

### Docker Images

Image layers are managed by Docker itself, not by the launcher. Use `docker images` and `docker system prune` to inspect or reclaim space.

---

## 12. Distribution: `.dtpkg` Packages

A `.dtpkg` file is a self-contained archive of one DT Type. Use it to share a scenario between teammates or to seed a fresh installation with a reference example.

### What a `.dtpkg` Includes

- The DT Type metadata (name, description, status).
- All Instance Types.
- All Instances (state + baseline + manual_position + physical_dimensions + parent hierarchy).
- All Algorithms (Python source).
- All Message Definitions (schemas).
- All Devices (synthetic and physical).
- The DT Type root floorplan (image + dimensions).
- Per-device floorplans (image + child positioning).
- All uploaded Instance Type icons referenced by `image_2d_path`.

### Exporting

In a DT Type's detail page, use **Export → Download .dtpkg**. The frontend prompts for a download location and saves the archive locally.

### Importing

Use the **Import .dtpkg** action. Importing is two-stage:

1. **Preview**: the backend extracts the archive into a staging directory, validates it, and returns a manifest plus a list of naming conflicts and a `pkg_id`. Nothing is committed yet.
2. **Commit**: with the `pkg_id` from the preview, the backend copies icon and floorplan files into their permanent locations and creates the DT Type with all of its configuration restored.

If a DT Type with the same name already exists, the preview surfaces the conflict so you can rename the existing DT Type, delete it, or abort the import before committing.

---

## 13. Algorithm Context API Reference

Every algorithm receives a `context` dict. The exact contents depend on the algorithm type. Below is the complete authoritative list — do not assume helpers exist beyond what is listed here.

### 13.1 The Device-Instance Link

A **device is a data source** (sensor, simulator, controller). An **instance is a digital twin object** (a forklift, a container, a building). They are linked through `dt_instance_devices`, a junction table.

**Cardinality rule:**
- One device belongs to at most **one instance** (its "owner"). The reverse lookup always returns 0 or 1 result.
- One instance can own **many devices** (e.g. GPS + battery + RFID + weight sensor).

**Two link types:**
- **Slot link** (formal): the instance template declares typed slots (e.g. `Forklift` requires a slot named `gps` with `device_type_id` pointing at `gps_sensor` in the device catalog). Slot fillers are declared during instance creation or via the Manage Devices modal.
- **Extra link** (free): for ad-hoc cases not covered by the template. No `slot_name`, just an association.

When an algorithm runs, the runtime resolves the source device's owner instance (if any) and injects extra fields into `context` so algorithms do not need to look up the owner manually.

### 13.2 Built-in Context Fields (All Algorithms)

| Field | Type | Purpose |
| --- | --- | --- |
| `context["device_id"]` | str | Identifier of the source device. |
| `context["device_name"]` | str | Human-readable name of the device. |
| `context["device_context"]` | dict | The `device_context` JSON field of the device (per-device configuration). |
| `context["instance"]` | dict or None | Full owner-instance row (`id`, `instance_id`, `instance_type`, `consolidated_state`, `slot_name`, …). `None` if the device has no owner. |
| `context["instance_id"]` | str or None | User-defined identifier of the owner instance. |
| `context["instance_uuid"]` | str or None | UUID of the owner instance. |
| `context["instance_state"]` | dict | `consolidated_state` of the owner (empty dict if no owner). |
| `context["devices"]` | dict | `{slot_name: device_dict}` — auxiliary devices in the same owner instance, keyed by slot. |

**Generation-only fields:**

| Field | Purpose |
| --- | --- |
| `context["last_execution"]` | Timestamp of the previous execution (or `None` on the first run). |
| `context["execution_count"]` | Integer counter, increments on every execution. |
| `context["message_definition"]` | Dict with `name` and `schemas` of the Message Definition this algorithm produces. |
| `context["params"]` | Dict passed by the user when triggering a manual generation algorithm via **Run now**. Empty for periodic generators. |

**Processing-only fields:**

| Field | Purpose |
| --- | --- |
| `context["timestamp"]` | Timestamp of the incoming message in ISO-8601 format. |

If the device has no owner instance, `context["instance"]` is `None` and the algorithm should fall back to payload-based logic. This is the standard pattern for observer devices (e.g. a gate RFID reader that observes many entities without "belonging" to any of them).

### 13.3 Implicit Helpers (Act on the Owner Instance)

These helpers operate on `context["instance"]` automatically — no instance identifier required. They become **no-ops** if the device has no owner.

| Helper | Use |
| --- | --- |
| `context["update_state"](patch)` | Shallow-merge `patch` into the owner instance's `consolidated_state`. To delete a key, pass `None` as its value. |
| `context["set_position"](x, y, z=0.0)` | Set `manual_position` of the owner. |
| `context["update_parent"](parent_identifier)` | Re-parent the owner (pass `None` to detach). |
| `context["reset_state"]()` | Restore the owner from its baseline (`manual_position`, `consolidated_state`, `parent_id`). |

Use these when the algorithm naturally manipulates only its own owner — the common case for telemetry algorithms on a single-owner device.

### 13.4 Explicit Helpers (Act on Any Instance by Identifier)

For cross-instance operations (e.g. a forklift updating the container it is carrying).

| Helper | Use |
| --- | --- |
| `context["get_instance"](identifier)` | Returns the full Instance dict (`consolidated_state`, `manual_position`, `physical_dimensions`, `parent_id`, `baseline_state`, `template`, …). Returns `None` if not found. |
| `context["get_instances_by_template"](template_name)` | Returns a list of Instances whose `instance_type` matches the given template name. |
| `context["update_instance_state"](identifier, patch)` | Shallow-merge `patch` into `consolidated_state`. To delete a key, pass `None`. Example: with current state `{"status": "idle", "battery": 80}` and patch `{"status": "moving", "battery": None}`, the result is `{"status": "moving"}`. |
| `context["update_instance_position"](identifier, x, y, z=0.0)` | Sets the Instance's `manual_position`. Visible in the Space View on the next polling tick. |
| `context["update_instance_parent"](identifier, parent_identifier)` | Re-parents an Instance. Pass `None` to detach. |
| `context["reset_instance_to_baseline"](identifier)` | Restores a single instance from its baseline. |

**`identifier` dual form.** Every helper that takes `identifier` accepts either the user-defined string id (e.g. `"container-a-1"`) or the raw UUID. Mixing forms within an algorithm is fine — the helper resolves which one it got.

### 13.5 Device-Level Helpers

For algorithms that manipulate the **source device itself** (geometry, color, persisted context) rather than its owner instance.

| Helper | Use |
| --- | --- |
| `context["get_device"](device_id)` | Returns the full Device dict (`device_mode`, `device_type_id`, `device_type_name`, `device_context`, `parent_id`, `geometry`, …). Returns `None` if not found. |
| `context["update_position"](device_id, x, y, z=0.0)` | Move the device marker on the floorplan (meters). |
| `context["update_floorplan_position"](device_id, x, y)` | Normalized position on its parent floorplan (0.0–1.0). |
| `context["set_color"](device_id, color)` | Recolor the device marker (hex string, e.g. `"#ff4d4f"`). |
| `context["set_geometry"](device_id, geometry)` | Atomic update of multiple geometry fields (`{x, y, z, color}` dict). |
| `context["update_device_context"](device_id, patch)` | Shallow-merge `patch` into the device's `device_context` JSON field. |

**`device_mode` vs `device_type_id`.** Devices have two distinct type fields, do not confuse them:
- `device_mode` — `"physical"` or `"synthetic"`. The runtime origin (real-world device vs simulated).
- `device_type_id` — UUID into `dt_device_types` (the catalog). The semantic category (`rfid_scanner`, `forklift_telemetry`, …). The catalog name is exposed as `device_type_name` for display.

### 13.6 Side-Effects: Alerts and Webhooks

| Helper | Use |
| --- | --- |
| `context["alert"](severity, message)` | Push a row to `dt_alerts`. `severity` is one of `"info"`, `"warning"`, `"critical"`. Alerts surface in the global Alerts tab and the header badge. Available in **both** generation and processing algorithms. Append-only — there is no "resolve" API. |
| `context["webhook"](url, payload, headers=None, timeout=5)` | One-shot HTTP POST to an external endpoint. Returns the parsed JSON response (or `{}` on success without body). Use for ERP integrations, Slack/Discord notifications, dashboards. |

```python
def process_data(data, context):
    if data.get("temperature", 0) > 90:
        context["alert"]("critical", f"Overheating: {data['temperature']}°C")
        context["webhook"]("http://erp.example/api/incidents", {
            "asset": context["instance_id"],
            "type": "overheat",
            "value": data["temperature"],
        })
    return data
```

### 13.7 State Ownership

**As of v2.9.6, the runtime no longer auto-merges outgoing telemetry into the owner instance's `consolidated_state`.** State is owned exclusively by the algorithm — if you want a telemetry field to land in state, write it yourself using `update_state` (implicit) or `update_instance_state` (explicit).

Telemetry is still persisted to the per-DT-Type message history for the Live Feed and the Dashboard, regardless of whether you wrote it to state.

This change avoids a subtle race: prior versions silently overwrote helper-written state with whatever the algorithm returned from `generate_data`, which made it impossible to keep a state field divorced from the telemetry payload (e.g. a status field whose value should not echo the latest telemetry verbatim).

**Return value contract:**
- Generation algorithms return a `dict` (the telemetry payload to publish), `None` to skip the tick, or a `list[dict]` to emit several messages.
- Processing algorithms return `dict` — the (possibly transformed) `data` — so a pipeline of processing algorithms can chain.
- Returning a non-dict raises an error in the message processor.

### 13.8 Example: Forklift Telemetry (with Owner Instance)

A `Forklift` instance owns a GPS device. The GPS device runs a generation algorithm that publishes position and updates the forklift's state implicitly.

```python
def generate_data(context):
    me = context["instance"]                         # the forklift
    if not me:
        return None                                  # device has no owner — skip
    state = me["consolidated_state"]

    next_x, next_y = compute_movement(state)         # your domain logic

    # Update implicitly — no instance ID needed
    context["set_position"](next_x, next_y)
    context["update_state"]({"speed": state.get("speed", 0) + 1})

    return {
        "forklift_id": me["instance_id"],
        "position": {"x": next_x, "y": next_y},
        "timestamp": context["timestamp"] if "timestamp" in context else None,
    }
```

If the algorithm needs to update a **different** instance (e.g. the container being carried), use the explicit helper:

```python
context["update_instance_position"](carried_container_id, next_x, next_y, next_z + 1.5)
```

### 13.9 Example: Observer Device (No Owner)

A gate RFID reader does not "belong" to a single container — it observes many. Such devices have no owner instance, so `context["instance"]` is `None` and the algorithm uses the payload to identify the observed entity.

```python
def process_data(data, context):
    seen_tag = data["payload"]["rfid_tag"]
    target = context["get_instance"](seen_tag)
    if target:
        context["update_instance_state"](seen_tag, {
            "last_seen": context["timestamp"],
            "last_gate": context["device_id"],
        })
    return data
```

### 13.10 Patterns and Tips

- **Prefer implicit helpers** (`update_state`, `set_position`) when working on the owner instance — less boilerplate, fewer chances to mistype an identifier.
- **Use explicit helpers** when the algorithm naturally touches multiple instances (carrier-and-cargo, sender-and-receiver, scanner-and-target).
- **Always read state via `get_instance`** inside the same tick rather than caching across ticks: another algorithm may have mutated it.
- **`update_state` and `update_instance_state` are shallow merges** — they avoid clobbering fields written by other algorithms. Replace `consolidated_state` wholesale only when you really want to reset it.
- **Normalize `manual_position`** when reading via `get_instance`: it may be `None`, a string (legacy JSON), or a dict. Defensive code: `pos = inst.get("manual_position") or {}`.
- **Helpers fail soft.** Each helper catches HTTP errors and returns `False` / `None` instead of raising, so a single network blip will not crash an algorithm. The error is `print()`-logged inside the DT container — grep for `[helper]` in the logs.
- **`webhook` is the escape hatch** for external integrations: anything you cannot do with the built-in helpers can be done via an HTTP webhook to a service you control.
- **Standard library only.** Algorithms run in a sandboxed Python environment — `datetime`, `json`, `math`, `random`, `statistics`, `urllib.request`, `uuid`, `re`, `collections` are available. Third-party packages (`numpy`, `pandas`, `requests`) are not.

---

## 14. Reproducibility: Baselines and Resets

A core feature of the framework is the ability to run the same scenario multiple times from a known starting state. This is achieved through baselines and resets.

### 14.1 Baseline State

Every Instance has a `baseline_state` field, automatically populated at creation time with a snapshot of:
- `manual_position`
- `consolidated_state`
- `parent_id`

The user can re-save the baseline at any time:
- **Per Instance**: in the Marker Popup or Edit Instance modal, click **Save as baseline**. Captures the current state of that single Instance.
- **All at once**: in the DT Instances tab header, click **Save baseline (all)**. Captures every Instance's current state in one operation.

### 14.2 Reset to Baseline

- **Per Instance**: in the Marker Popup or Edit modal, click **Reset to baseline**. Restores `manual_position`, `consolidated_state`, and `parent_id` for that single Instance.
- **All at once**: in the DT Instances tab header, click **Reset to baseline (all)**. Restores every Instance.

Resetting preserves the hierarchy and is the recommended way to start a new run of an experiment.

### 14.3 Reset Experiment

In the Space View, the **Reset** button performs a different operation: it clears `consolidated_state` for every Instance (without restoring positions or parents). Useful when you want to keep the spatial layout but rerun an algorithm from a clean state.

### 14.4 Recommended Workflow for Reproducible Experiments

1. Create your DT Type and model the entities.
2. Position everything correctly.
3. Click **Save baseline (all)** to lock the initial state.
4. Run your scenario (manual triggers, periodic algorithms, etc.).
5. Inspect Live Feed, Alerts, Telemetry to validate.
6. Click **Reset to baseline (all)** to restart from the same initial conditions.
7. Repeat with different parameters to compare.

---

## 15. REST API and External Integration

The backend exposes a REST API on `http://localhost:3000/api/`. Every action available in the frontend is also available over HTTP.

### Common Endpoints

| Endpoint | Purpose |
| --- | --- |
| `GET /api/digital-twin-types` | List all DT Types. |
| `POST /api/digital-twin-types` | Create a DT Type. |
| `GET /api/digital-twin-types/{id}/instances` | List Instances of a DT Type. |
| `POST /api/digital-twin-types/{id}/instances` | Create an Instance. |
| `PATCH /api/digital-twin-types/{id}/instances/{iid}/position` | Update an Instance's position. |
| `POST /api/digital-twin-types/{id}/instances/save-baseline-bulk` | Save baseline for all instances. |
| `POST /api/digital-twin-types/{id}/instances/reset-to-baseline-bulk` | Reset all instances to baseline. |
| `POST /api/digital-twin-types/{id}/reset-experiment` | Clear consolidated_state for all instances. |
| `POST /api/algorithms/{id}/execute-generation` | Test mode (returns generated payloads). |
| `POST /api/algorithms/{id}/trigger` | Real trigger of a manual generation algorithm. |
| `GET /api/digital-twin-types/{id}/algorithms` | List algorithms. |
| `GET /api/alerts` | Read alerts. |
| `POST /api/export/preview` | Dry-run that returns size and counts for an export request without writing bytes. |
| `POST /api/export` | Build and stream a `.dtpkg` archive. The request body selects which DT Types and options. |
| `POST /api/import/preview` | Upload a `.dtpkg`, extract it to a server-side staging folder, and return a manifest + a list of naming conflicts plus a `pkg_id`. |
| `POST /api/import/commit` | Commit a previously previewed import using the `pkg_id`. |

### Webhooks

Algorithms can call external services via the `webhook` helper. The framework itself does not currently expose webhook receivers other than the MQTT bridge.

### MQTT

Physical devices publish messages to MQTT topics that Bridge subscribes to. The exact topic per device is shown in the Message Definitions tab. Use this to integrate real hardware (industrial sensors, IoT devices, message brokers).

---

## 16. Glossary

- **Algorithm**: a Python function executed by the DT container; either generates or processes messages.
- **Baseline state**: a snapshot of an Instance's `manual_position`, `consolidated_state`, and `parent_id` used by Reset operations.
- **Bridge**: the service that ingests external (MQTT) messages and routes them to the right DT container.
- **Color rules**: per-template conditional rules that override an Instance's marker color based on `consolidated_state` fields.
- **Consolidated state**: the live JSON state field on each Instance, freely shaped by algorithms.
- **Data Flow**: a binding on a Device that connects either a generation algorithm (synthetic) or a Message Definition (physical) to a processing pipeline. Required for any data to actually move; periodic generators tick only while bound to an active Data Flow.
- **Device**: a concrete instance of a Device Type, either synthetic or physical.
- **Device Slot**: a named binding between an Instance Type and an expected Device category.
- **Device Type**: a template describing what kind of messages a device produces or consumes.
- **DT container**: a per-DT-Type Docker container that runs that scenario's algorithms.
- **DT Type**: the top-level scenario container.
- **Enricher**: the service that decorates raw telemetry with metadata before storing.
- **Execute (button)**: test-mode button that runs an algorithm and returns the result without emitting through the live pipeline.
- **Floorplan**: an image representing the spatial layout of a DT Type or Device, with known real-world dimensions.
- **Instance**: a concrete entity (a row in the database) of a given Instance Type.
- **Instance Type**: a template describing the visual and structural defaults of a kind of entity.
- **Layer order**: a per-template z-index controlling rendering and hit-test order in the 2D viewer.
- **Live Feed**: real-time stream of every message produced inside a DT Type.
- **Manual position**: the explicit (x, y, z) location of an Instance, in meters, used by the visualizer.
- **Message Definition**: a JSON schema (inferred from an example) describing one type of message.
- **Physical dimensions**: the (width, height, depth) of an Instance in meters, used to size the marker on the map.
- **Run now (button)**: real trigger button for `trigger_type='manual'` generation algorithms.
- **Stack**: a collection of Instances at the same X/Y with different Z values; rendered as a single marker with a count badge.
- **Synthetic device**: a Device whose data source is an algorithm running inside the DT container (no external feed).
- **Webhook**: HTTP POST to an external URL, available to algorithms as `context["webhook"]`.
