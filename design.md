# ICS 3202: Artificial Intelligence — Practical Exercise 3
**Topic:** Problem-Solving Agents (Autonomous Self-Driving Trailer Agent)  
**Deliverable File:** `design.md`  
**Target Repository:** `PS-Agents`  

---

## Task 1: Agent Design using PEAS Framework & Environment Properties

### 1.1 PEAS Framework Representation

| Component | Detailed Specification |
| :--- | :--- |
| **Performance Measure** | Safety (zero collisions, zero damage to cargo or road infrastructure), trip time minimization, fuel/energy economy, strict compliance with Kenyan traffic laws (speed limits, lane discipline), ride comfort, trailer cargo stability, and path cost optimization. |
| **Environment** | Kenyan national trunk highways (A104, B1, C67, A2, etc.), surrounding traffic (matatus, commercial haulage trucks, saloon cars, motorcycles/boda-bodas), pedestrians, domestic and stray livestock, changing weather patterns (heavy tropical downpours, fog across the Rift Valley escarpment, mud), road topography, variable pavement quality (speed bumps, potholes, unpaved diversions), and roadside physical signage. |
| **Actuators** | Electronic steering motor, electronic drive-by-wire throttle, pneumatic trailer dual-circuit braking system, automated transmission gear selector, turn indicators and emergency hazard lights, high/low beam headlights, audible warning horn, and in-cabin telemetry communication display. |
| **Sensors** | 360-degree LiDAR scanner arrays, stereoscopic forward/rear/lateral optical cameras, long/medium-range radar units, Differential GNSS/GPS navigation receiver, Inertial Measurement Unit (IMU - gyroscopes and accelerometers), wheel rotation speed sensors, electronic axle suspension load sensors, trailer kingpin hitch articulation angle encoders, and onboard powertrain diagnostics. |

---

### 1.2 Five Environment Characteristics (Definitions of Agent Environments)

1. **Partially Observable (vs. Fully Observable):**
   * The trailer agent cannot perceive the entire global state of the environment at any given instant. Obstructions caused by other long vehicles, blind spots around the trailer body, terrain geometry (e.g., hairpin bends down the Rift Valley escarpment), and adverse weather phenomena (thick fog in Limuru or torrential rain) restrict the perceptual range of onboard sensors.
2. **Stochastic (vs. Deterministic):**
   * The subsequent state of the operating environment is not entirely determined by the agent's current actions. Random interventions—such as erratic overtaking by third-party drivers, livestock crossing the carriageway, mechanical tire punctures, and unexpected road blockages—introduce non-deterministic transitions.
3. **Sequential (vs. Episodic):**
   * Actions executed in early driving stages directly affect all future states. Decisions regarding speed regulation when approaching steep descents, route branch selection at highway junctions, and following distance accumulation influence fuel reserves, brake wear, trailer stability, and long-term travel duration.
4. **Dynamic (vs. Static):**
   * The driving environment undergoes continuous autonomous changes while the agent deliberates. Pedestrians, traffic streams, and weather conditions continue evolving irrespective of whether the trailer is in motion or stationary.
5. **Continuous (vs. Discrete):**
   * Fundamental environmental dimensions—such as vehicle velocity, longitudinal and lateral acceleration, steering angle, inter-vehicle distance, and temporal progression—range across continuous real-number spectra rather than discrete grid cells or turns.
6. **Multi-Agent (vs. Single-Agent):**
   * The agent operates in a populated environment alongside competitive and collaborative entities, including human motorists, pedestrian traffic, traffic police marshals, and animal road users, whose behaviors demand game-theoretic and predictive interaction modeling.

---

## Task 2: Highway Network Directed Graph, Search Tree, and Abstraction

### 2.1 Concept of Abstraction Applied
To construct a computationally tractable discrete state space from continuous geographical terrain:
* **State Abstraction:** The infinite space of physical GPS coordinates along the Kenyan road network is abstracted into a finite set of discrete topological transit nodes representing major junctions and towns (Nairobi, Naivasha, Nakuru, Kericho, Kisumu, etc.).
* **Action Abstraction:** Continuous vehicle maneuvers (lane centering, throttling, pedal actuation, braking) are abstracted into high-level discrete navigation transitions: `Traverse_Link(Town_Source, Town_Destination)`.
* **Path Cost Abstraction:** Multidimensional physical expenses (fuel volume, brake pad friction, tire wear, elevation delta) are abstracted into a single, uniform scalar cost function: **road travel distance in kilometers ($g(n)$)** along designated highway corridors.

---

### 2.2 Directed Highway Network Graph

Below is the topological representation of the transit network spanning Nairobi, Central Highlands, the Rift Valley, and the Lake Victoria Basin:

```
                     [Nyeri]
                    /       \
              85 km/         \105 km
                  /           \
           [Thika]             [Nanyuki]
              |                    |
         42 km|                    |160 km
              |                    |
[Nairobi] ----+---- 88 km ----> [Naivasha] ---- 68 km ----> [Nakuru]
                                                              |
                          +-----------------------------------+-----------------------------------+
                          | 110 km                                                            | 140 km
                          v                                                                   v
                     [Kericho]                                                           [Eldoret]
                     /       \                                                           /       \
               87 km/         \112 km                                              64 km/         \120 km
                   /           \                                                       /           \
                  v             v                                                     v             v
             [Kisumu]        [Kisii]                                              [Kapsabet]     [Kitale]
                                \                                                     |
                                 \115 km                                         73 km|
                                  \                                                   v
                                   +--------------------------------------------> [Kisumu]
```

#### Adjacency List with Link Distances:
* `Nairobi -> Naivasha`: 88 km (A104)
* `Nairobi -> Thika`: 42 km (A2)
* `Thika -> Nyeri`: 85 km (A2)
* `Nyeri -> Nanyuki`: 105 km (A2)
* `Nanyuki -> Naivasha`: 160 km (C77)
* `Naivasha -> Nakuru`: 68 km (A104)
* `Nakuru -> Kericho`: 110 km (B1)
* `Nakuru -> Eldoret`: 140 km (A104)
* `Kericho -> Kisumu`: 87 km (B1)
* `Kericho -> Kisii`: 112 km (B3)
* `Kisii -> Kisumu`: 115 km (A1)
* `Eldoret -> Kapsabet`: 64 km (C39)
* `Kapsabet -> Kisumu`: 73 km (C39/C37)
* `Eldoret -> Kitale`: 120 km (A1)

---

### 2.3 Search Tree (Root: Nairobi, Target Goal: Kisumu)

The tree illustrates node expansions evaluated by accumulated path cost $g(n)$:

```
                                [Nairobi (g=0)]
                                  /         \
                            88 km/           \42 km
                                /             \
                     [Naivasha (g=88)]       [Thika (g=42)]
                            |                      |85 km
                       68 km|                  [Nyeri (g=127)]
                            v                      |105 km
                     [Nakuru (g=156)]         [Nanyuki (g=232)]
                       /          \                |160 km
                110 km/            \140 km    [Naivasha (g=392)]
                     /              \              ...
            [Kericho (g=266)]      [Eldoret (g=296)]
              /          \              /         \
        87 km/     112 km \       64 km/           \120 km
            /              \          /             \
   [KISUMU (g=353)]   [Kisii (g=378)] [Kapsabet (g=360)] [Kitale (g=416)]
       (GOAL)               |              |73 km
                     115 km |       [KISUMU (g=433)]
                            v            (GOAL)
                      [KISUMU (g=493)]
                           (GOAL)
```

---

## Task 3: Optimal Path Determination (Nairobi to Kisumu)

### 3.1 Path Cost Evaluations

| Route Designation | Trajectory Sequence | Segment Distances (km) | Total Cost $g(n)$ |
| :--- | :--- | :--- | :--- |
| **Route A (Mau Summit / Kericho Trunk)** | Nairobi $\rightarrow$ Naivasha $\rightarrow$ Nakuru $\rightarrow$ Kericho $\rightarrow$ Kisumu | $88 + 68 + 110 + 87$ | **353 km** |
| **Route B (North Rift / Kapsabet Corridor)** | Nairobi $\rightarrow$ Naivasha $\rightarrow$ Nakuru $\rightarrow$ Eldoret $\rightarrow$ Kapsabet $\rightarrow$ Kisumu | $88 + 68 + 140 + 64 + 73$ | **433 km** |
| **Route C (South Nyanza / Kisii Circuit)** | Nairobi $\rightarrow$ Naivasha $\rightarrow$ Nakuru $\rightarrow$ Kericho $\rightarrow$ Kisii $\rightarrow$ Kisumu | $88 + 68 + 110 + 112 + 115$ | **493 km** |
| **Route D (Central Aberdare Loop)** | Nairobi $\rightarrow$ Thika $\rightarrow$ Nyeri $\rightarrow$ Nanyuki $\rightarrow$ Naivasha $\rightarrow$ Nakuru $\rightarrow$ Kericho $\rightarrow$ Kisumu | $42 + 85 + 105 + 160 + 68 + 110 + 87$ | **657 km** |

---

### 3.2 Algorithmic Execution Trace (Uniform Cost Search)

* **Step 1:** Frontier: `[(Nairobi, 0)]`
* **Step 2:** Expand `Nairobi (0)` $\rightarrow$ Frontier: `[(Thika, 42), (Naivasha, 88)]`
* **Step 3:** Expand `Thika (42)` $\rightarrow$ Frontier: `[(Naivasha, 88), (Nyeri, 127)]`
* **Step 4:** Expand `Naivasha (88)` $\rightarrow$ Frontier: `[(Nyeri, 127), (Nakuru, 156)]`
* **Step 5:** Expand `Nyeri (127)` $\rightarrow$ Frontier: `[(Nakuru, 156), (Nanyuki, 232)]`
* **Step 6:** Expand `Nakuru (156)` $\rightarrow$ Frontier: `[(Nanyuki, 232), (Kericho, 266), (Eldoret, 296)]`
* **Step 7:** Expand `Nanyuki (232)` $\rightarrow$ Frontier: `[(Kericho, 266), (Eldoret, 296), (Naivasha, 392 - closed)]`
* **Step 8:** Expand `Kericho (266)` $\rightarrow$ Frontier: `[(Eldoret, 296), (Kisumu, 353), (Kisii, 378)]`
* **Step 9:** Expand `Eldoret (296)` $\rightarrow$ Frontier: `[(Kisumu, 353), (Kapsabet, 360), (Kisii, 378), (Kitale, 416)]`
* **Step 10:** Pop `Kisumu (353)` from frontier. Goal criteria met. Terminate search.

### 3.3 Optimal Route Summary
* **Optimal Path:** **Nairobi $\rightarrow$ Naivasha $\rightarrow$ Nakuru $\rightarrow$ Kericho $\rightarrow$ Kisumu**
* **Total Shortest Distance:** **353 km**

---

## Evidence of Groupwork & Discussion Minutes

### Team Minutes of Meeting
* **Course Unit:** ICS 3202: Artificial Intelligence
* **Session Date:** October 16, 2026
* **Session Platform / Location:** Physical Lab Session / Google Meet
* **Meeting Objective:** Formulation, Agent Design, and Problem-Solving Graph Modeling for Exercise 3

#### Members in Attendance:
1. Kubai Dan - 190226
2. Sydney Wairimu - 192135
3. Jibril Jaafar - 191831

#### Meeting Proceedings:
1. **PEAS Formulation:** The team formulated the performance criteria, sensory devices, actuators, and physical driving conditions tailored to autonomous trailer operation along Kenyan trunk highways.
2. **Environment Classification:** Defined the operational context across the five core Russell & Norvig dimensions: partially observable, stochastic, sequential, dynamic, and continuous.
3. **Graph Topology & Abstraction:** Defined discrete state nodes for key transit hubs connecting Nairobi, Central Highlands, Rift Valley, and Lake Victoria Basin, abstracting highway routes using Google Maps distance references.
4. **Search Tree & Cost Optimization:** Formulated the tree rooted at Nairobi and executed an algorithmic trace of Uniform Cost Search to mathematically confirm the shortest path to Kisumu (353 km via Kericho).

#### Group Discussion & Collaboration Evidence:
Below is the virtual session screenshot showing active group deliberation and screen-sharing of `design.md` on VS Code via Google Meet:
<img width="1418" height="866" alt="team_meeting" src="https://github.com/user-attachments/assets/4dad546f-d154-478e-b4eb-e38d31f45ae6" />

![Group Discussion Evidence](./team_meeting.png)

*Caption: Collaborative review session on Google Meet showing active participation and live document walkthrough.*
