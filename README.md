# MANET Attack Simulation — Black Hole & Worm Hole vs Normal AODV

A complete NS-3 simulation framework for comparative performance analysis of
**Black Hole** and **Worm Hole** attacks on the AODV routing protocol in MANETs.

**Based on:**
- Al Rubaiei et al., *"Performance Analysis of Black Hole and Worm Hole Attacks in MANETs"*, IJCNIS, 2022
- Bishnoi & Mandoria, *"Performance Study of Black Hole Attack Detection using AODV in MANET"*, IJCSE, 2015

---

## Project Structure

```
manet-attack-sim/
├── ns3/
│   ├── manet-attack-sim.cc      # Main NS-3 simulation (all 3 scenarios)
│   ├── aodv-blackhole.h         # Black Hole AODV: derived routing protocol header
│   └── aodv-blackhole.cc        # Black Hole AODV: implementation
├── python/
│   ├── parse_flowmon.py         # FlowMonitor XML parser -> CSV metrics
│   ├── plot_results.py          # matplotlib comparison plots
│   └── plot_results.gp          # gnuplot alternative
├── results/                     # Output directory (auto-created)
│   ├── results.csv              # Aggregated simulation results
│   ├── flowmon-*.xml            # Per-run FlowMonitor output
│   └── *.png                    # Generated plots
├── run_all_simulations.sh       # Master batch runner
└── README.md                    # This file
```

---

## Network Topology (from Reference Papers)

| Parameter         | Value                          |
|-------------------|--------------------------------|
| Terrain           | 1500m x 300m                   |
| Total nodes       | 50                             |
| Malicious nodes   | 0-10 (swept)                   |
| Mobility model    | Random Waypoint                |
| Max speed         | 10 m/s, pause=0s               |
| MAC protocol      | IEEE 802.11b Ad Hoc            |
| Routing protocol  | AODV (RFC 3561)                |
| Traffic           | CBR/UDP, 5 flows, 500B packets |
| Simulation time   | 90 seconds                     |
| Propagation       | Friis (omni-directional)       |

---

## Prerequisites

### NS-3 (3.40+)

```bash
wget https://www.nsnam.org/releases/ns-allinone-3.43.tar.bz2
tar xjf ns-allinone-3.43.tar.bz2 && cd ns-allinone-3.43
./build.py --enable-examples --enable-tests
cd ns-3.43 && ./ns3 run hello-simulator   # verify
```

### Python

```bash
pip install matplotlib pandas numpy
```

---

## Installation

```bash
NS3_ROOT=~/ns-allinone-3.43/ns-3.43
cp ns3/manet-attack-sim.cc  $NS3_ROOT/scratch/
cp ns3/aodv-blackhole.h     $NS3_ROOT/scratch/
cp ns3/aodv-blackhole.cc    $NS3_ROOT/scratch/
cd $NS3_ROOT && ./ns3 build
```

---

## Running Simulations

### Manual single run

```bash
cd $NS3_ROOT

# Scenario 0: Normal AODV
./ns3 run "scratch/manet-attack-sim --scenario=0 --maliciousCount=0 --seed=1 --verbose"

# Scenario 1: Black Hole, 5 malicious nodes
./ns3 run "scratch/manet-attack-sim --scenario=1 --maliciousCount=5 --seed=1 --verbose"

# Scenario 2: Worm Hole, 4 malicious nodes
./ns3 run "scratch/manet-attack-sim --scenario=2 --maliciousCount=4 --seed=1 --verbose"
```

### Full automated sweep

```bash
chmod +x run_all_simulations.sh
./run_all_simulations.sh ~/ns-allinone-3.43/ns-3.43
# Runs ~60 simulations, writes results/results.csv, generates plots
```

---

## Attack Implementations

### Black Hole Attack

**How it works:**
1. Malicious node receives RREQ broadcast
2. Sends fake RREP immediately with DestSeqNo=UINT32_MAX (appears "freshest") and HopCount=1
3. Source node routes all traffic through malicious node (highest seq# wins in AODV)
4. Malicious node drops every forwarded packet

**NS-3 implementation:** `BlackHoleRoutingProtocol` (derived from `aodv::RoutingProtocol`):
- `RecvRequest()` overridden -> calls `SendFakeRrep()` instead of normal processing
- `RouteInput()` overridden -> silently drops all forwarded (non-local) packets

### Worm Hole Attack

**How it works:**
1. WH-Entry node captures RREQs and tunnels them to WH-Exit
2. WH-Exit rebroadcasts as if it originated them (appears 1 hop from source)
3. Nodes build routes through the wormhole pair
4. Data packets are dropped by both wormhole nodes

**NS-3 implementation:** UDP tunnel on port 9999 between the two colluding nodes via `WormHoleTunnelApp`. Both nodes have packet sinks installed to absorb (drop) all forwarded data.

---

## Performance Metrics

| Metric | Formula |
|--------|---------|
| PDR | `(Rx packets / Tx packets) x 100` |
| E2E Delay | `delaySum / Rx packets` (ms) |
| NRL | `(Tx - Rx) / Rx` |
| Throughput | `(Rx bytes x 8) / simTime / 1000` (kbps) |

---

## Generating Plots

```bash
# From real simulation results:
python3 python/plot_results.py --input results/results.csv --output-dir results/

# Demo mode (no NS-3 needed):
python3 python/plot_results.py --demo --output-dir results/

# PDF output for publication:
python3 python/plot_results.py --input results/results.csv --format pdf

# gnuplot alternative:
gnuplot python/plot_results.gp
```

**Generated files:**
- `comparison_all_metrics.png` — 2x2 grid of all four metrics
- `metric_PDRpct.png`, `metric_E2EDelayms.png`, `metric_NRL.png`, `metric_Throughputkbps.png`
- `bar_snapshot.png` — bar chart at selected malicious counts

---

## Expected Results (reference paper baseline)

| Metric      | Normal AODV | Black Hole (m=5) | Worm Hole (m=5) |
|-------------|-------------|------------------|-----------------|
| PDR         | ~95%        | ~55-65%          | ~65-75%         |
| E2E Delay   | ~45 ms      | ~110-130 ms      | ~115-140 ms     |
| NRL         | ~0.12       | ~0.35-0.45       | ~0.40-0.55      |
| Throughput  | ~185 kbps   | ~115-135 kbps    | ~145-160 kbps   |

**Key findings:** Black Hole degrades PDR faster (complete drop). Worm Hole introduces higher E2E delay before dropping. Both attacks significantly increase NRL due to AODV route repair cycles.

---

## Troubleshooting

- **Build error** — Ensure all 3 files are in `<ns3-root>/scratch/`
- **PDR=0** — Run with `--verbose` to check flow setup and IP assignment
- **Wormhole no effect** — Needs `maliciousCount >= 2`; try more seeds for averaging
- **FlowMonitor empty** — Check simulation start times; flows begin at t=2s
