# About VirtusCo

**VirtusCo** is a robotics startup based in Thripunithara, Kerala, India, building autonomous porter robots that transform baggage handling in airports and beyond.

**Mission:** Help travelers move through large airports effortlessly by providing smart, autonomous luggage assistance.

**Vision:** Democratizing robotics -- bridging the gap between those with resources and those without, while building tailored robotic solutions for any industry.

| | |
|---|---|
| **Website** | [virtusco.in](https://virtusco.in) |
| **Email** | virtusco.tech@gmail.com |
| **GitHub** | [github.com/VirtusCo](https://github.com/VirtusCo) |
| **LinkedIn** | [linkedin.com/company/virtusco](https://www.linkedin.com/company/virtusco/) |

---

## Founding Team

| Name | Role | Focus Area |
|------|------|------------|
| **Antony Austin** | Founder | Software architecture, ROS 2, ESP32 firmware, AI, system integration |
| **A. Azeem Kouther** | Founder | Mechanical design, hardware assembly, motor systems |
| **Allen George Thomas** | Founder | Industrial design, CAD, UX, touchscreen UI |
| **Alwin George Thomas** | Founder | Financial strategy, investment planning, revenue modelling |
| **Danush Krishna** | Founder | Electronics design, PCB layout, sensor wiring, power management |

---

## The Product -- Porter (Virtus)

An autonomous luggage-carrying robot for airports. Passengers interact via a touchscreen. Porter follows them through the terminal carrying their bags, providing flight info, wayfinding, and check-in assistance.

### Differentiation

| Advantage | Detail |
|-----------|--------|
| **120 kg payload** | 2.4x Incheon Airporter, 10x Travelmate |
| **Airport integration** | Ticket and boarding pass scanning for gate-aware navigation |
| **Built-in weighing scale** | Baggage compliance at the robot |
| **LiDAR + sensor fusion** | 360 LIDAR + triple-sensor Kalman filter |
| **Price** | ~$8,400 per unit vs $30,000+ competitors |
| **No cloud dependency** | All processing runs on-device |

### Target Markets

1. **Airports** (primary)
2. Cruise ships
3. Hotels and resorts
4. Restaurants
5. General autonomous goods transport

### Business Model

Direct B2B sales to airports with recurring software and maintenance revenue.

| Item | Value |
|------|-------|
| Selling price | INR 7 Lakh per unit |
| Manufacturing cost | INR 4 Lakh per unit |
| Gross margin | ~43% |
| Software subscription | INR 40,000/year per unit |
| Annual maintenance | INR 70,000/year per unit |

---

## Traction

- **Pilot interest** from Kempegowda International Airport (T2, Bangalore) and Maldives International Airport
- **Survey validation:** 63% of frequent flyers said "Yes, absolutely!" to using a robot porter; 48% would pay INR 300--500 per use
- **Awards:** Two-time winner of "Best Entrepreneur" competition + multiple startup accelerator successes

---

## Stage

Pre-seed, pre-funding. Building a working prototype to demonstrate to investors. The software is the longest-lead item -- hardware is assembled or procurable.

### Expansion Phases

| Phase | Scope |
|-------|-------|
| Phase 1 | Kerala airports + 1--2 early adopters |
| Phase 2 | Major Indian hubs: BLR, DEL, BOM |
| Phase 3 | South Asia, GCC, and global |

---

## Engineering Culture

This is a safety-critical product operating around passengers in airports. Code is production-grade from day one -- tested, documented, and reviewable.

| Principle | Implementation |
|-----------|---------------|
| Fail-safe on any error | Sensors fail or comms lost -- stop motors immediately |
| Emergency stop always available | Physical e-stop + software e-stop service |
| Graceful degradation | Single sensor failure -- continue with reduced capability |
| No silent failures | Every error logged, published to `/diagnostics` |
| Passenger safety first | Conservative speed limits, wide obstacle margins |
