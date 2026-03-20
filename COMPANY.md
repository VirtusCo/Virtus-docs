# COMPANY.md — VirtusCo Context for Claude

> **This file gives Claude (our AI engineer) full context on who we are, what we're building, and why.**
> Claude should read this to understand business context, team dynamics, and product decisions.
>
> Last updated: 28 Feb 2026

---

## 1. About VirtusCo

**VirtusCo** is a robotics startup based in **Thripunithara, Kerala, India** building
autonomous porter robots that transform baggage handling in airports and beyond.

**Mission:** We help travelers move through large airports effortlessly by providing smart, autonomous luggage assistance.

**Vision:** Democratizing robotics — bridging the gap between those with resources and those without, while building tailored robotic solutions for any industry.

**Website:** [virtusco.in](https://virtusco.in)
**Email:** virtusco.tech@gmail.com
**Phone:** +91 73568 44578

### Social & Code

| Platform | Link |
|----------|------|
| GitHub (company) | [github.com/VirtusCo](https://github.com/VirtusCo) |
| GitHub (Porter repo) | [github.com/austin207/Porter-ROS](https://github.com/austin207/Porter-ROS) |
| LinkedIn | [linkedin.com/company/virtusco](https://www.linkedin.com/company/virtusco/) |
| X / Twitter | [x.com/VirtuscoTech](https://x.com/VirtuscoTech) |
| Instagram | [instagram.com/virtuscotech](https://www.instagram.com/virtuscotech/) |

---

## 2. Founding Team

| Name | Role | Expertise | Focus Area on Porter |
|------|------|-----------|---------------------|
| **Antony Austin** | Founder | Autonomous systems, embedded systems, AI | Software architecture, ROS 2 stack, ESP32 Zephyr firmware, system integration. Primary dev directing Claude. |
| **A. Azeem Kouther** | Founder | Mechanical systems, hardware | Mechanical design, hardware assembly, motor systems |
| **Allen George Thomas** | Founder | Product design, UX, CAD | Industrial design, CAD models, user experience, touchscreen UI |
| **Alwin George Thomas** | Founder | Financial strategy, investment planning | Business model, revenue modelling, investor relations |
| **Danush Krishna** | Founder | Electronics hardware | Electronics design, PCB layout, sensor wiring, power management, circuit integration |

### How the Team Works

- **Antony** is the primary person directing Claude. He reviews all code Claude generates, tests on hardware, and makes architecture decisions.
- **Danush** handles electronics hardware (PCBs, wiring, power, sensor circuits) — Claude may need to respect pinouts and electrical constraints he communicates.
- **Azeem** builds the physical robot — Claude needs to respect mechanical constraints (mount positions, wiring, weight limits) when they're communicated.
- **Allen** designs the product experience — Claude may be asked to build the touchscreen UI to his specifications.
- **Alwin** handles business side — unlikely to interact with Claude directly, but business deadlines drive engineering priorities.

### Claude's Role in the Team

Claude is the **primary software engineer** for the ROS 2 stack. Responsibilities:

1. **Write all driver code** (C++ LIDAR driver, Python processors, bridge nodes)
2. **Write all infrastructure** (Docker, CI/CD, launch files, configs)
3. **Write tests** (unit, integration, lint)
4. **Write documentation** (READMEs, guides, changelogs)
5. **Advise on architecture** when asked (protocols, libraries, patterns)
6. **Produce complete, buildable, tested code** — not snippets or pseudocode

Antony reviews everything. Code must be production-grade from day one — this is a startup product, not a hobby project.

---

## 3. The Product — Porter Robot

### What It Is

An **autonomous luggage-carrying robot** for airports. Passengers interact via a touchscreen. Porter follows them through the terminal carrying their bags, providing flight info, wayfinding, and check-in assistance along the way.

### Target Markets

| Priority | Market | Use Case |
|----------|--------|----------|
| 1 (primary) | **Airports** | Carry passenger luggage through terminals |
| 2 | Cruise ships | Cabin luggage transport |
| 3 | Hotels & resorts | Guest luggage, room service delivery |
| 4 | Restaurants | Food/order delivery between kitchen and tables |
| 5 | General | Any venue needing autonomous goods transport |

### Key Product Capabilities

| Feature | Description |
|---------|-------------|
| **120 kg payload capacity** | Robust chassis for multiple heavy bags, stable while manoeuvring |
| **Autonomous + Semi-Autonomous modes** | Dual-mode: fixed-path autonomous + semi-auto for busy terminals |
| **LiDAR obstacle avoidance** | 360° LIDAR + triple-sensor fusion (ToF + Ultrasonic + Microwave via Kalman filter) |
| **Airport system integration** | Ticket & boarding pass scanning for gate-aware navigation |
| **Built-in weighing scale** | Baggage compliance check before check-in |
| **Interactive display** | Touchscreen: route, check-in counter, boarding gates, flight status, airport maps |
| **Adjustable lifting mechanism** | Smart lift for various luggage sizes; ground ↔ platform ↔ counter |
| **ESD-safe design** | Electrostatic discharge protection for operational safety in airport environments |
| **Hands-free luggage transport** | Passengers walk freely while Porter carries their bags |

### Product Name

**Virtus** — "Meet Virtus, the Self-Driving Luggage Porter for Airports."

### Business Model

**Direct B2B Sales** to airports, with recurring software + maintenance revenue.

| Item | Value |
|------|-------|
| **Selling Price** | ₹7 Lakh per unit |
| **Manufacturing Cost** | ₹4 Lakh per unit |
| **Gross Profit per Unit** | ₹3 Lakh (~43% margin) |
| **Software Subscription** | ₹40,000/year per unit |
| **Annual Maintenance Contract (AMC)** | ₹70,000/year per unit |
| **Total ARR per Unit** | ₹1.1 Lakh/year |

### 3-Year Financial Forecast

| Year | Units Sold | Total Revenue | Total Gross Profit |
|------|-----------|---------------|-------------------|
| Year 1 | 10 | ₹81 Lakh | ₹30 Lakh |
| Year 2 | 30 | ₹2.43 Crore | ₹90 Lakh |
| Year 3 | 75 | ₹6.075 Crore | ₹2.25 Crore |

### Market Opportunity

| Market | Size | Notes |
|--------|------|-------|
| **TAM (Global)** | $12–16 Billion by 2030 | Global airport automation, 6.2% CAGR |
| **Airport Robots** | $3.5–3.7 Billion by 2030 | 16–18% CAGR |
| **SAM (India)** | ~$500M | India aviation: USD 14.8B (2025) → USD 26B+ (2030), 10–12% CAGR. 300–600M air passengers by 2030. |
| **SOM (Initial)** | $50–100M | Kerala airports (CIAL, Trivandrum, Calicut) + top hubs |

**Early adopter airports investing in smart tech:** BLR, HYD, CHN, CIAL.

### Survey Validation (Frequent Flyers)

| Question | Result |
|----------|--------|
| Would pay ₹300–₹500 for the service | **48.1%** |
| "Yes, absolutely!" to using a robot | **63%** |
| Consider luggage assistance "very important" | **55.6%** |

---

## 4. Hardware Specification

### System Architecture Diagram

```
┌────────────────────────────────────────┐   ┌─────────────────────┐
│  Obstacle Avoidance Sensor Fusion      │   │ Rich Environmental  │
│  ┌─────────┐                           │   │    Information      │
│  │  ToF     │──┐                       │   │                     │
│  ├─────────┤  │  ┌────────────┐        │   │  ┌───────────────┐  │
│  │Ultrasonic│──┼─►│  Kalman    │──► ESP32 ──┼──►│ Raspberry Pi  │◄─┼── LIDAR
│  ├─────────┤  │  │  Filter    │        │   │  │  (Master)     │  │  (YDLIDAR
│  │Microwave │──┘  └────────────┘        │   │  │               │  │   X4 Pro)
│  └─────────┘                           │   │  │ SLAM, Nav2,   │  │
└────────────────────────────────────────┘   │  │ State Mgmt,   │  │
                                             │  │ Display       │  │
                                             │  └───────┬───────┘  │
                                             └──────────┼──────────┘
                                                        │
                                             ┌──────────▼──────────┐
                                             │    Motor Control     │
                                             │                      │
                                             │       ESP32          │
                                             │      ┌──┴──┐        │
                                             │  ┌───▼──┐ ┌▼────┐   │
                                             │  │BTS7960│ │BTS7960│  │
                                             │  └───┬──┘ └┬────┘   │
                                             │  ┌───▼──┐ ┌▼────┐   │
                                             │  │12V DC│ │12V DC│   │
                                             │  │100RPM│ │100RPM│   │
                                             │  │40Kgcm│ │40Kgcm│   │
                                             │  └──────┘ └──────┘   │
                                             └──────────────────────┘
```

### Component Inventory

| Component | Spec | Quantity | Role |
|-----------|------|----------|------|
| **Raspberry Pi** | RPi 4 (current) / RPi 5 (upgrade target) | 1 | Master compute — ROS 2, Nav2, SLAM, display |
| **YDLIDAR X4 Pro 360°** | 360° scanner, serial USB, baudrate 128000 | 1 | Primary navigation sensor, SLAM input |
| **ESP32** | ESP32 DevKit | 2 | Slave microcontrollers |
| **BTS7960** | 43A dual H-bridge motor driver | 2 | Drive motor power stage |
| **DC Motor** | 12V, 100 RPM, 40 Kgcm torque | 2 | Differential drive (left + right) |
| **ToF Sensor** | Time-of-Flight distance sensor | 1+ | Short-range obstacle detection |
| **Ultrasonic Sensor** | HC-SR04 or similar | 1+ | Mid-range obstacle detection |
| **Microwave Sensor** | RCWL-0516 or similar | 1+ | Motion/presence detection (through materials) |
| **Lifting Mechanism** | Motor/actuator (TBD by Azeem) | 1 | Luggage height adjustment |
| **Touchscreen Display** | Connected to RPi (HDMI or DSI) | 1 | Passenger interaction |

### Drive System

- **Type:** Differential drive (2 powered wheels + casters)
- **Motors:** 2× 12V DC, 100 RPM, 40 Kgcm torque
- **Drivers:** 2× BTS7960 dual H-bridge (43A capable)
- **Encoders:** Not installed yet — firmware and ROS code must support future encoder feedback for odometry
- **Control:** RPi sends velocity commands → ESP32 #1 → BTS7960 PWM → Motors

### Sensor Fusion Strategy

The three environment sensors serve complementary roles:

| Sensor | Best At | Weakness | Range |
|--------|---------|----------|-------|
| **Time-of-Flight** | Precise short-range distance | Narrow FOV, ambient light sensitive | 0.02–4m |
| **Ultrasonic** | Mid-range, works in dark/bright | Soft surfaces absorb sound, wide cone | 0.02–4m |
| **Microwave** | Detects through materials, motion | Not precise distance, binary presence | 1–7m |

**Fusion method:** Kalman filter on ESP32 #2 combines all three into a unified obstacle/environment estimate before sending to RPi. This offloads processing from the RPi and provides richer data than any single sensor.

### Communication

| Link | Protocol | Direction | Data |
|------|----------|-----------|------|
| LIDAR → RPi | Serial USB (direct) | LIDAR → RPi | Raw scan data |
| ESP32 #2 → RPi | USB CDC Serial | ESP32 → RPi | Fused sensor data (distances, obstacle classification) |
| RPi → ESP32 #2 | USB CDC Serial | RPi → ESP32 | Config commands (thresholds, enable/disable) |
| RPi → ESP32 #1 | USB CDC Serial | RPi → ESP32 | Velocity commands (linear x, angular z), lift commands |
| ESP32 #1 → RPi | USB CDC Serial | ESP32 → RPi | Motor status, encoder data (future), lift position |

---

## 5. Current Stage & Business Context

### Where We Are

**Pre-seed, pre-funding.** Building a working prototype to demonstrate to investors.

The prototype must prove:
1. The robot can navigate autonomously in an indoor space
2. It can carry luggage stably
3. It avoids obstacles (people, pillars, carts, walls)
4. The touchscreen provides useful passenger info
5. It's reliable enough for a 5-minute demo loop

### Why Speed Matters

- No external funding yet — the team is self-funding
- A working prototype is the **gate** to investor conversations
- The software is the longest-lead item — hardware (chassis, motors, sensors) is assembled or procurable
- Claude writing production-grade code at speed is a **strategic advantage** for the startup

### Competitors

| Competitor | Product | Cost | Payload | Weakness |
|-----------|---------|------|---------|----------|
| **Travelmate** | Follow-me suitcase | $600 | 11 kg | Low capacity, not airport-integrated |
| **Incheon "Airporter"** | Airport robot | $30,000 | 50 kg | Extremely expensive |
| LG CLOi, various pilots | Enterprise robots | Very high | Varies | Expensive, not scalable for smaller airports |

### VirtusCo Differentiation

| Advantage | Detail |
|-----------|--------|
| **Airport-system integration** | Ticket & boarding pass scanning → gate-aware navigation |
| **Built-in weighing scale** | Baggage compliance at the robot, before check-in |
| **120 kg payload** | 2.4× Incheon Airporter, 10× Travelmate |
| **LiDAR obstacle avoidance** | 360° LIDAR + triple-sensor fusion |
| **ESD-safe design** | Operational safety in high-traffic airport environments |
| **Dual-mode operation** | Fixed-path autonomous + semi-autonomous |
| **Price** | ₹7 Lakh (~$8,400) — fraction of $30K competitors |

### Traction

- **Pilot interest:** Early-stage pilot deployment enquiries from **Kempegowda International Airport (T2, Bangalore)** and **Maldives International Airport**
- **Product maturity:** Engineered a scalable industrial-level robot for air delivery service providers
- **Awards:** Two-time winner of "Best Entrepreneur" competition + multiple startup accelerator successes

### Other Technical Projects (Team Portfolio)

- **Indoor Air Flow Measurement Robot:** Semi-autonomous system using ESP32, high-torque planetary motors, and custom IR sensors to measure HVAC efficiency
- **Robotic Arm:** ESP32 + MATLAB controlled, GoogleNet-based transfer learning for automated object classification, pick-and-place

### Business Expansion Phases

| Phase | Scope |
|-------|-------|
| **Phase 1** | Geographic expansion: Kerala airports + 1–2 early adopters |
| **Phase 2** | Major Indian hubs: BLR, DEL, BOM |
| **Phase 3** | South Asia, GCC, and global players |

**Growth strategy:** Software value integration, airport system partnerships, robot variants for different payloads, scaling to subscription-based model to reduce airport CAPEX.

---

## 6. Engineering Culture & Principles

### Code Quality

This is a **safety-critical product** operating around passengers (children, elderly, disabled) in airports. Code must be:
- **Production-grade from day one** — no "we'll fix it later" tech debt
- **Tested** — unit tests, integration tests, lint
- **Documented** — every node, parameter, topic, launch file documented
- **Reviewable** — clear commits, changelogs, PR descriptions

### Safety Philosophy

| Principle | Implementation |
|-----------|---------------|
| **Fail-safe on any error** | If LIDAR drops, sensors fail, or comms lost → stop motors immediately |
| **Emergency stop always available** | Physical e-stop button + software e-stop service |
| **Graceful degradation** | If one sensor type fails, continue with reduced capability + alert |
| **No silent failures** | Every error logged, published to `/diagnostics`, visible on display |
| **Passenger safety first** | Conservative speed limits, wide obstacle margins, stop-before-collision |

### Development Workflow

1. Antony describes what's needed
2. Claude writes complete, buildable code
3. Antony reviews and tests on hardware
4. Iterate until it works correctly
5. Commit with proper conventional commit message + CHANGES.md entry
6. Move to next task

### Communication Style with Claude

- Be direct and technical — no hand-holding needed
- Produce complete files, not snippets
- When unsure, state assumptions and ask
- When something seems wrong in the instructions, flag it
- Prioritise working code over perfect code, but never skip tests or docs

---

## 7. Key Dates & Milestones

| Date | Milestone |
|------|-----------|
| 25 Feb 2026 | Hardware investigation, architecture decision, repo setup |
| 28 Feb 2026 | Claude.md, OBJECTIVES.md, COMPANY.md, skills/ — project fully documented |
| ~14 Mar 2026 | **Phase 1 deadline:** LIDAR driver publishing `/scan` from real hardware |
| ~21 Mar 2026 | Phase 2: Nav2 autonomous navigation working |
| ~28 Mar 2026 | Phase 3: ESP32 bridge nodes, motor control via ROS 2 |
| ~Apr 2026 | Phases 4–6: Orchestration, display, cross-compilation |
| ~May 2026 | Phases 7–8: OTA, security, simulation |
| ~May/Jun 2026 | **Phase 9: MVP — investor demo ready** |

---

## 8. For Claude: What You Need to Know

### You are building software for a real physical robot that:
- Operates in **airports** around **real passengers**
- Carries **heavy luggage** (safety and stability matter)
- Must be **reliable** — a crash/freeze during an investor demo kills funding prospects
- Runs on **constrained hardware** (Raspberry Pi, not a server)

### Your code will:
- Run on a real RPi controlling real motors via real ESP32s
- Parse real serial data from a real LIDAR spinning on a real robot
- Feed real Nav2 which drives a real robot through real spaces
- Be shown to real investors deciding whether to fund this startup

### So:
- **Handle every error.** Log it, recover from it, never crash.
- **Be efficient.** RPi has limited CPU/RAM. Don't waste either.
- **Be modular.** Hardware will change (LIDAR model, motor type, sensors). Config changes, not code changes.
- **Be testable.** If it can't be tested without hardware, add mock/simulation support.
- **Be documented.** The next engineer (human or AI) should understand everything from the docs.
