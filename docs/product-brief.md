# Product brief: SDV Fleet Operations & Hazard Monitor

## Goal

A fleet operator watches a small fleet of electric delivery vans in Oshawa, Ontario. Each van detects its own driving hazards and sends them to the cloud. The operator sees each hazard on a live map within 2 seconds, acknowledges it, and compares how safe and efficient each route is.

## Fleet

Three simulated battery-electric vans, each driving its own route in a loop:

| Van | Id | Route | Speed limit |
|---|---|---|---|
| Van 01 | `veh-01` | A: North (Taunton Rd) | 60 km/h |
| Van 02 | `veh-02` | B: Downtown (King St) | 50 km/h |
| Van 03 | `veh-03` | C: East (Harmony Rd) | 70 km/h |

The vans and routes come from one configuration file. Adding a van means adding one entry there.

## Vehicle side

- **Simulator:** drives each van along its route and produces its signals: position, speed, acceleration, traction control state, battery state of charge, coolant temperature, odometer and trip number. Runs are repeatable with a seed, and a time scale speeds them up.
- **Digital twin:** each van's signals live in an Eclipse Kuksa Databroker, using standard VSS (Vehicle Signal Specification) names such as `Vehicle.Speed`. Fleet and hazard signals that VSS doesn't have go in a custom VSS overlay.
- **Hazard detector:** reads the van's signals from Kuksa, applies the rules below, and writes the most severe current hazard back into Kuksa with the time it was detected (t0).
- **Publisher:** sends each van's telemetry and hazards to the cloud.
- **Software management:** Eclipse Ankaios starts each van's workloads (databroker, simulator, detector, publisher) from one manifest and restarts any that stop.

### Hazard rules

Severity: 1 warning, 2 high, 3 critical. When several rules fire, the most severe hazard wins.

| Hazard | Fires when | Severity |
|---|---|---|
| Harsh braking | deceleration ≥ 4 m/s² | high |
| Harsh braking | deceleration ≥ 7 m/s² | critical |
| Traction loss | traction control active at ≥ 20 km/h | critical |
| Overheat | coolant > 90 °C | high |
| Low energy | battery ≤ 20% | warning |
| Low energy | battery ≤ 10% | high |
| Overspeed | speed > route limit + 10 km/h | warning |

Thresholds are configurable. The rules have no I/O so they can be tested on their own.

## Transport

Vehicle to cloud goes over Eclipse Zenoh, one key for telemetry and one for hazards per van. The vehicle and cloud code never import each other; they share only the message keys and the message schema.

## Cloud data platform

- Telemetry and hazard events enter **Google Pub/Sub**.
- An **Apache Beam** pipeline (runs locally with the DirectRunner, deploys to **Dataflow**) validates the messages, drops or quarantines malformed ones, enriches them (van name, route, latency), and writes them to storage.
- **Storage** keeps telemetry, alerts and completed trips.
- Message schemas are written down as contracts, and there is reproducible sample input data for testing the pipeline.

## Cloud alerts and analytics

- A van that sends no telemetry for 10 seconds is marked offline and raises an alert.
- **Trip results:** when a van finishes a trip, record distance, energy used (a rise in battery means charging, not negative use), number of hazards, and hazards per 100 km.
- **Latency:** t1 (cloud receipt) − t0 (detection). Report median, p95 and max. Target: p95 under 2 seconds.

## Dashboard

A web dashboard, updated live:

- **Map** of Oshawa with the three vans moving on their routes, and a dot per hazard coloured by severity (yellow warning, orange high, red critical). A van with an active hazard pulses red.
- **Fleet panel:** speed, battery, trip number and odometer per van.
- **Alert queue:** newest first, each with its latency. The operator can acknowledge an alert and choose to show resolved ones.
- **Latency panel:** median, p95 and max against the 2 second target.
- **Route performance table:** filled in after each van's first full trip. Route B (downtown) should show the most hazards per 100 km.

## Running it

- **Laptop demo:** one command starts the three vans, the cloud side and the dashboard in one process, with stand-ins for Kuksa, Zenoh and the cloud services so no infrastructure is needed. Options: inject N hazards for a latency test, port, time scale, seed, use the real Kuksa.
- **Cloud deployment:** the vans run on a GCP VM under Ankaios; the cloud side runs on GCP (GKE for the service and dashboard, Pub/Sub and Dataflow for the pipeline).
- **CI/CD:** every pull request runs the tests. The main branch deploys automatically.

## Quality

- Unit tests for the rules, analytics and pipeline transforms; integration tests with the real Kuksa, Zenoh and Pub/Sub emulator; an end-to-end test from a hard-braking signal on a van to an alert in the cloud in under 2 seconds.
- Failure cases: malformed messages, a van going offline, a broker that isn't reachable.
- Tests say clearly whether they run against stand-ins or real services.
