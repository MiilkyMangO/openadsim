# Benchmark scenarios on the OpenADSim map `campus`

Scenarios for comparing a driver that uses only its own sensor with one that also receives V2X messages (CAM). Each of them has a visibility gap: something relevant is hidden from the ego's sensor. Classification of every file: [`splits.csv`](splits.csv).

Terms: **L1** = the ego's own sensor only. **L2** = own sensor plus CAM. **Stack** = the default OpenADStack. **Lane-entry TTA** = the time the ego still needs to reach the pedestrian when the pedestrian enters the ego lane.

## The four scenario types

| Scenario | Situation | What it tests | Can V2X close the gap? |
|---|---|---|---|
| AccidentTwoWays | An accident group blocks the ego's lane on a road with one lane per direction; oncoming cars come at intervals, hidden by the tallest vehicle of the group | Whether a driver starts to overtake while a hidden car approaches | Yes, the oncoming cars can send CAM |
| OppositeVehicleTakingPriority | The ego turns right at a junction where it has to give way; a parked van hides a car T coming from the left | Whether a driver gives way to a car it cannot see in time | Yes, T can send CAM |
| ParkingCrossingPedestrian | A pedestrian walks out from in front of a parked car, across the ego lane | The reaction to a hidden pedestrian | **No: control scenario.** The pedestrian sends no message, so the gap exists and V2X cannot close it |
| OccludedHardBrake | A car F, ahead of the lead car L, brakes hard; L hides F and reacts later | Whether a driver gets the warning before L brakes | Yes, F can send CAM; L1 learns of the emergency when L brakes, L2 when F brakes |

## Dev and test

| Classification | Meaning |
|---|---|
| `dev` | May be seen and rerun while developing a driver. The same position with changed parameters |
| `test` | A new position or a new geometry. **Not looked at during development**; run only when the driver is final. A test scenario whose result has been seen counts as dev. No test scenarios are in this folder yet |
| `critical` | A dev scenario whose outcome differs between identical runs on the stack. **Repeat it several times and report the ratio**; a single run does not represent it |
| `limit` | Practically every reactive driver fails, so it does not suit developing a driver; only for a specific question, such as anticipation. None in this folder yet |

## How values are chosen

Values are chosen for a **rule-following driver**, not for one particular stack: at the speed limit (13.89 m/s), with a cautious driver at 8.33 m/s beside it; in OccludedHardBrake, the reference follower keeps a distance of half the speed in km/h, in metres ('halber Tacho'; 19.8 m at 11 m/s). The stack only verifies: its speed profile predicts a run and takes no part in the choice.

A scenario is accepted when it runs and the intended situation arises. How a particular driver then behaves is a result, not a condition. Each variant was run once before being added, except `target1389`, which ran twice; the delivered files ran more often.

## Files

The R numbers point to the results register of the thesis this library was built for, which is not public. All values in the table are from those runs and design calculations.

| File | Changed from the delivered scenario | Why | Classification | R |
|---|---|---|---|---|
| `campus_accident_two_ways.xosc` | Delivered: oncoming cars at 13.89 m/s | Road 93 | dev | R-030 |
| `campus_opposite_vehicle_taking_priority.xosc` | Delivered: T at 16.67 m/s, trigger at ego front 11 m before the mouth | junction221 | dev | R-043, R-044 |
| `campus_parking_crossing_pedestrian.xosc` | Delivered: walker at 2 m/s, TTC condition value 3.6 s | Road 21 | dev | R-066 |
| `campus_occluded_hard_brake.xosc` | Delivered: L reacts 1.0 s after F | Road 64 | dev | R-080 |
| `variants/accident_two_ways/oncoming1111.xosc` | Oncoming cars at 11.11 m/s (40 km/h) | Slower traffic, hidden longer: measured 3.40-3.45 s against about 2.7 s for cars 2-8 | dev | R-085 |
| `variants/opposite_vehicle_taking_priority/target1389.xosc` | T at 13.89 m/s (the speed limit), trigger at 14.5 m | T keeps the limit. The earlier trigger keeps the conflict, but T is first visible for 11-12 frames, then hidden 0.95-1.05 s. **Repeat several times and report the contact ratio:** of two runs on the stack, one ended in contact and one yielded | **critical** | R-086, R-087 |
| `variants/parking_crossing_pedestrian/ttc33.xosc` | TTC condition value 3.3 s | A harder trigger: lane-entry TTA 1.22 / 1.24 s at 13.89 / 8.33 m/s, against 1.52 / 1.54 s. The ego arrives at 67 % of the period in which the walker is within its width, more central than 87 % | dev | R-088, R-089 |
| `variants/parking_crossing_pedestrian/walker14.xosc` | Walker at 1.4 m/s, TTC condition value 4.4 s | Normal walking, about the same difficulty: lane-entry TTA 1.46 / 1.48 s. The walker is visible earlier (TTA 3.07 s against 2.62 s). The trigger lies close to the start of the road, so the 13.89 m/s reference is idealised there | dev | R-088, R-089 |
| `variants/occluded_hard_brake/tau15.xosc` | L reacts 1.5 s after F | A larger difference in information time between L1 and L2. L stops 3.50 m behind F | dev | R-083 |

## Using the files

The files are imported already and carry their criteria: **do not import them again.** Set `MAP=campus`, `SCENARIO_FILE` to the file and `SPAWN_POINT` as follows.

| Scenario | `SPAWN_POINT` |
|---|---|
| AccidentTwoWays | `-144.406,-291.653,263.474,111.498` |
| OppositeVehicleTakingPriority | `105.022,289.456,250.02,-90.21` |
| ParkingCrossingPedestrian | `-176.618,-207.184,262.271,-74.25` |
| OccludedHardBrake | `294.146,-285.601,253.572,-90.13` |

Contacts are evaluated independently of the runner's collision criterion, for example by the overlap of the bounding boxes. The pedestrian type comes from the runner's seed: check `type_id` after a run; these files expect `walker.pedestrian.0020`.
