# Research Plan: Independent Validation of Open-Source AGV Safety

Drafted 2026-11-08. Status: plan only, nothing built or measured yet.

## Goal

Test Polymath Robotics' open-source safety work from outside the company, on a different machine, and publish the evidence. Two outputs from the same work:

1. **Community contribution:** test reports, scripts, and issues or PRs upstream; evidence filed with the Open Source Safety Consortium (opensourcesafe.com).
2. **IEEE paper**, if the results hold up. The content-audit session decides relevance and venue (see Open questions).

## Paper claim (revised 2026-11-08 after the IEEE brainstorming session's review)

**Verdict:** a paper only with WS4. WS1–WS3 on their own confirm a spec, so reviewers would read them as a test report. They go upstream as consortium evidence.

> A localization-health monitor built on stock slam_toolbox and EKF signals detects degradation within X s at Y false stops per hour. Combined with the measured stop-chain latency, it bounds travel after a fault to Z m.

The novelty is the **end-to-end budget**: fault, then detection, then stop request, then robot stopped, all measured on open hardware. It is not new integrity math.

- WS4 supplies the research claim (the detection terms X and Y).
- WS1 supplies the latency term in the budget.
- WS2, WS3 and WS5 are community evidence and get one paragraph in the paper.

**Positioning:** practical, not theoretical. It uses off-the-shelf SLAM signals with no integrity-risk derivation, wired into an open stop chain and measured end to end. **Never claim** a "first localization integrity monitor for AGVs": prior work exists (see Prior work).

## Prior work (checked 2026-11-08 by web search; read the full papers before citing)

- **Localization integrity for ground robots (confirmed):** Joerger, Duenas Arana, Spenko and Pervan adapt aviation integrity risk to lidar navigation, covering data-association risk and unmapped obstacles (Sensors, 2018). Duenas Arana, Abdul Hafez, Joerger and Spenko, "Recursive Integrity Monitoring for Mobile Robot Localization Safety" (2019), is a mobile-robot version. Hassani et al. add IMU and lidar intensity. Joerger is now at Virginia Tech. Review-supplied links still to open: par.nsf.gov/biblio/10547583, par.nsf.gov/biblio/10203876, repository.iit.edu islandora:1024309, publikationen.badw.de/de/050323929.
- **Localizability and degeneracy detection (confirmed, two venue corrections):**
  - Nubert, Walther, Khattak and Hutter, "Learning-based Localizability Estimation for Robust LiDAR Localization", **IROS 2022** (not ICRA), best student paper finalist. arXiv 2203.05698.
  - Tuna et al., "X-ICP: Localizability-Aware LiDAR Registration for Robust Localization in Extreme Environments", **IEEE T-RO vol. 40, 2024**, pp. 452–471, DOI 10.1109/TRO.2023.3335691 (not 2023).
  - Alsayed et al., "Failure Detection for Laser-based SLAM in Urban and Peri-Urban Environments", ITSC 2017. Works on raw 2D scans, so it does not depend on the SLAM method. **Closest baseline for WS4.**
  - Alsayed, Resende and Bradai, "Operational Design Domain Monitoring at Runtime for 2D Laser-Based Localization Algorithms", ICAR 2021. **Very close to WS4; must cite and compare.** It shows ICAR accepts this topic.
  - Use these as **monitor baselines** and compare against at least one, preferably Alsayed 2017 or 2021 (both 2D lidar, like the RPLIDAR).
- **Standards (partly checked):** the current edition is ISO 3691-4:2023 (2020 withdrawn), and a revision (ISO/DIS) is in enquiry. A secondary index says it covers navigation, obstacle detection and operational design domains. **Not verified:** whether it ties localization loss to a required safety function. The standard is paywalled, so read it through a university library or a standards body before writing the motivation paragraph.
- **Independent validation of open-source safety components:** essentially nothing peer-reviewed. The novelty holds but is thin on its own.
- **ORBES tie:** the ORBES negative result (drift during fix gaps can't be observed, and controllers on the drifted estimate fail) is the failure mode WS4 detects. Cite ORBES once if it is public by then. Do not merge the two papers.

## Testbed

| Item | What | Status |
|---|---|---|
| Robot | Hiwonder-style mecanum kit: Raspberry Pi 5, RPLIDAR (likely A2), Orbbec Astra Pro Plus, IMU on the motor controller, GL.iNet router. 3.5 kg, 0.4 m/s max. Details in the [Mecanum SLAM robot doc](https://claude.ai/artifact/PELQpbug1P9GucaTr2fnG1) | Unpowered. Blocked on 3S battery |
| Software | ROS 2 (version to confirm), slam_toolbox, robot_localization EKF, Nav2 | Not booted |
| Protective stop | ESP32-S3 remote and switch built from `polymathrobotics/protective-stop` open hardware; ROS 2 `machine_bridge_node` on the Pi | Not built. Get the bill of materials from the repo |
| Ground truth | Floor tape grid plus fiducial markers and an overhead camera (reuse ORBES fiducial work) | Not set up |
| Measurement | Logic analyzer or a second ESP32 timestamping the stop line and packets; MCAP logging | To buy or borrow |

Not covered by this testbed: radar (no radar on the robot, so `radar_layer` testing is out of scope unless one is bought), CAN and drive-by-wire (the kit uses USB serial).

### Testbed caveats (state these in the paper)

- **Small and slow:** 3.5 kg at 0.4 m/s stops almost instantly, so the fault-to-stop budget is mostly detection and network latency, with braking near zero. Industrial AGVs run at 1.5–2 m/s with hundreds of kg. Report the measured detection and latency terms, then extrapolate travel to industrial speeds and masses. Label it clearly as extrapolation.
- **Software-only stop path:** as shipped, "stop" means the Pi sending zero velocity to the motor controller over USB, so a hung Pi or controller could ignore it. Real AGVs cut power through a safety relay or a drive safety input.
- **Fix:** add a relay or contactor in the battery-to-controller line, held closed only while `machine_bridge_node` reports OK. Silence then physically removes motor power. Rate the relay for motor stall current (over 8 A briefly) and fit it after the 7.5 A fuse. Measure both paths: the software stop, and the stop when the relay drops.
- **Hobby-grade parts:** nothing is safety-rated. Results are test evidence only (see Ground rules).
- **Single platform:** see the Generality section under WS4.

## Workstreams

Ranked by value to the community and to the paper.

### WS1. Protective-stop timing under network faults (top priority)

- **Question:** Does the machine stop within heartbeat × missed windows (default 400 ms × 5 = 2.0 s) of the last good packet, across every link and fault?
- **Method:** Timestamp the last valid packet and the stop request. Sweep: link (Ethernet, USB-NCM, Wi-Fi), packet loss 0–50%, latency jitter, link failover mid-run, Tailscale tunnel cut, router power loss. Use `tc netem` on the Pi side to inject loss.
- **Metrics:** Distribution of silence-to-stop time (median, p99, max), failover gaps, any case where the robot keeps moving past the bound.
- **Context numbers:** at 0.4 m/s, 2.0 s of silence is 0.8 m of travel before the stop request (before braking). At industrial AGV speeds of 1.5–2 m/s it is 3–4 m. Shows why integrators must tune the timing.
- **Deliverable:** Latency dataset, test scripts, a report filed as consortium evidence.

### WS2. Fault injection on the dual-channel switch

- **Question:** Do the claimed failure behaviors hold: wire fault makes the two cores disagree, so nothing is sent; machine starts stopped; arming needs a 500 ms hold?
- **Method:** Open each normally closed contact separately, short across channels, add contact bounce, stall one core's task, corrupt one core's message, power-cycle mid-arm. Arming edge cases at 499 / 501 ms and double-taps. Check that a `stop_only` remote cannot arm.
- **Metrics:** Pass or fail per fault, compared with the draft safety case's claims.
- **Note:** Polymath already runs fault-injection, chaos and soak tests. Our value is that we are independent and on a different machine. Read their test suite first so we don't repeat it.

### WS3. Independent review of the safety case for the consortium

- **Question:** This answers the consortium's own open questions. What needs independent review, and how much safety evidence carries over to another machine?
- **Method:** Trace the protective-stop hazard analysis to its requirements, then to code and tests. List gaps and untested assumptions. Then adapt the evidence to the mecanum AGV and record each item as carried over, modified or redone.
- **Metrics:** Share of evidence items reused unchanged; hours spent versus starting from scratch.
- **Deliverable:** Review report and the AGV-specific safety argument. This is the first outside contribution to the consortium.

### WS4. Localization integrity tests with `replay_testing` (fits this folder's SLAM focus)

- **Question:** When SLAM degrades, does the robot know, and does it stop in time?
- **Method:** Record MCAP runs on the AGV, then use `polymathrobotics/replay_testing` to replay them with injected faults: lidar sector occlusion, lidar dropout, featureless corridor, kidnapped robot, mecanum wheel slip, and IMU bias. Build a localization health monitor (for example slam_toolbox match score, EKF covariance growth, scan-to-map residual). Feed it into `machine_bridge_node` as a stop source.
- **Metrics:** Pose error against ground truth; time from fault to detection; false-stop rate in clean runs; stop distance from fault onset.
- **Baselines:** compare the monitor against at least one published localizability or degeneracy detector (see Prior work).
- **Generality:** one mecanum robot is not enough for WS4. Options, cheapest first:
  1. Replay public real-world 2D-lidar datasets through the monitor with `replay_testing`. This is real data on other robots at no hardware cost.
  2. Simulation (Gazebo): repeatable fault injection and a ground truth for free. It supports the claims, but reviewers discount it for generality because sim lidar has no real glass, reflections or wheel slip. Use it next to real data, not instead of it.
  3. A second physical robot with a different drive and lidar (for example a differential-drive TurtleBot 4 borrowed from a university lab or maker space). Needed for ICRA or RA-L strength.

  Report mecanum wheel slip as a platform-specific fault, which helps the paper.
- **Deliverable:** Upstream replay tests for localization regression, and the health-monitor node. Links to the ORBES occlusion work.

### WS5. Reproduce `lidar_eval_framework` on the RPLIDAR

- **Question:** Do their lidar scores reproduce in another lab, and how does a low-cost 2D lidar score?
- **Method:** Run their framework on the RPLIDAR A2. Add conditions they may not cover: glass, shiny floors, dark targets, dust or fog spray.
- **Deliverable:** Results contributed to their public lidar dashboard. Separate from the paper unless it supports WS4.

### Smaller items (do when time allows)

- `livekit_ros2_bridge`: subscriptions drop on heartbeat loss; topic deny list cannot be bypassed.
- `event_recording`: pre-event buffer complete under CPU load on the Pi 5.

## Sequence and blockers

1. Buy the battery and inline fuse, then complete robot bring-up (robot doc checklist). **Blocks WS1, WS2 with the robot, WS4 and WS5.**
2. Build the protective-stop hardware. WS2 bench tests can start before the robot runs.
3. WS3 desk review can start now: it needs only the repo and docs.
4. Run WS1, then WS4, then combine them (localization monitor feeding the protective stop).
5. WS5 and the smaller items as time allows.

## Ground rules

- **Responsible disclosure:** report any safety defect to Polymath privately first. Publish after they have had time to respond.
- **No certification claims:** SIL 3 and PL e are Polymath's targets, not certifications. Our results are test evidence, not certification.
- **Not an emergency stop:** Polymath's repo calls the protective stop a controlled, preventive shutdown that is not designed to be an emergency stop. Nothing in the paper or reports may imply an e-stop or a safety-rated function.
- Contact the consortium before submitting the paper, so findings land upstream first.
- Treat every placeholder value from the robot doc as unmeasured until calibration replaces it.

## Venue options (deadlines checked 2026-11-08; most come from the IEEE RAS calendar or aggregators, so confirm on official sites)

| Venue | Tier | 2027 deadline | Fit |
|---|---|---|---|
| CASE 2027 | Mid | 1 Mar 2027 | Same day as IROS and ORBES; robot not ready. Skip |
| ICAR 2027 | Mid | 21 Mar 2027 | Published Alsayed 2021 on the same topic. Tight: needs WS4 data by March |
| ETFA 2027 | Mid, industrial | Not posted (ETFA 2026 was 19 Apr) | Safety-engineering angle; good if the monitor result is modest |
| ROSCon UK 2027 (Bristol, 5–7 Jul) / ROSCon Global 2027 (Krakow, 12–14 Oct, wiki only) | Community talk | Not posted | Consortium and open-source audience; not a paper |
| **ROBIO 2027** | Mid, broad | Not posted (ROBIO 2026 was 1 Aug 2026) | **Best timeline fit**: about 1 Aug 2027 leaves spring for bring-up and summer for data |
| ICRA 2028 (Guadalajara, about 15–19 May 2028, unconfirmed) | Top | Likely mid-Sept 2027 (ICRA 2027's was 15 Sept 2026) | Needs a second platform and baseline comparison |
| RA-L | Top journal | Rolling | Same bar as ICRA |

**Recommended path:**
1. arXiv preprint and a ROSCon talk with first WS1 and WS4 results.
2. ROBIO 2027, or ICAR 2027 if data is ready by March.
3. ICRA 2028 or RA-L only if the second-platform results are strong.

Not IROS 2027: it clashes with ORBES.

## Open questions

1. Which public 2D-lidar datasets suit replay of degradation faults?
2. Can a second robot, such as a TurtleBot 4, be borrowed?
3. Does the ISO 3691-4:2023 text really say nothing about localization loss? This needs access to the standard.
4. ROBIO or ICAR first? Decide by about Feb 2027, based on the robot bring-up date.

## Session split (to set up later)

Each can run as its own Claude Code session once the robot is up:

- WS3 desk review (can start now, read only)
- WS1 and WS2 protective-stop hardware and timing tests
- WS4 localization replay tests and health monitor
- WS5 lidar evaluation
- Paper drafting: workshop or arXiv version first, then ICRA 2028 or RA-L
