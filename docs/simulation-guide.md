# Simulation guide

This guide follows the behavior implemented in this checkout. The broader project goals in `.claude/CLAUDE.md` describe planned features; they are not all part of the running simulation.

## Start the app

From the repository root, run:

```sh
python main.py
```

`main.py` reads `config.yaml`. If it is missing, it writes a default file and exits. Otherwise, it checks that the configured GTFS directory exists, loads the configuration, creates the simulator, and starts the Pygame visualizer. The checked-in `config.yaml` points to `GTFS/`, requests U8, U2, and U5, and sets two trains per line, a one-second time step, 10 km/h train speed, 20-second station dwell, and 1,000 initial passengers. The required GTFS files are `stops.txt`, `routes.txt`, `trips.txt`, and `stop_times.txt` (see `src/data/gtfs_loader.py`). A missing GTFS directory or zero successfully loaded lines ends startup with an error. Individual line load failures are reported as warnings and skipped.

The visualizer runs at the configured FPS. Press Space to pause or resume, R to reset, and Q or Escape to quit (`src/visualization/pygame_visualizer.py`).

## Startup and simulation flow

`Config.from_yaml` in `src/core/config.py` builds validated simulation and visualization settings. `Simulator` (`src/simulation/simulator.py`) asks `GTFSLoader` for each configured route. It creates the configured number of trains on each loaded line, placing them at evenly spaced station indices, then spawns the configured initial passenger count. Each passenger is assigned a random loaded line and two distinct stations on that line; the passenger record starts at the chosen origin.

While the visualizer is unpaused, it calls `Simulator.step()` `int(simulation_speed)` times per frame. Each step updates every train by `time_step_seconds`, then advances the simulation clock by that amount. A traveling train advances by its speed converted from km/h to km/s. When it reaches the next station, the next update starts the fixed dwell timer; the train then counts it down and departs. At either end of the ordered line it reverses direction. The visualizer draws the lines, stations, waiting passenger markers, trains, and status UI. Station labels are always drawn using the first four characters of each station's `name`; the `show_station_names` setting is not consulted.

`simulation_speed` therefore controls the integer number of steps per rendered frame, with fractional values truncated. `reset()` sets the clock to zero and respawns trains; it does not recreate the initial passenger list, so passengers stay at their original stations while trains return to their configured starting positions.

## Core objects

- **`Station`** (`src/core/station.py`): immutable, validated station ID, names, and geographic coordinates.
- **`Line`** (`src/core/line.py`): immutable ordered stations, line speed, and display color; calculates distances between adjacent stations from coordinates.
- **`Train`** (`src/core/train.py`): mutable train state, including current and next station, segment progress, direction, dwell timer, capacity, and onboard passenger list.
- **`Passenger`** (`src/core/passenger.py`): passenger ID, origin, destination, and optional current station ID.
- **Configuration models** (`src/core/config.py`): `SimulationConfig` and `VisualizationConfig` validate their YAML settings; `Config` groups both sections.
- **`Simulator`** (`src/simulation/simulator.py`): loads lines, creates trains and initial passengers, advances train state and time, and supplies positions and statistics to the visualizer.

## Implemented behavior and TODOs

Implemented behavior includes GTFS route and station loading, train spawning and back-and-forth movement, fixed dwell times, initial random same-line passenger records, simulation time advancement, and Pygame rendering. Passenger records are displayed at stations, but they are not boarded or moved: the boarding/alighting point in `Train` is explicitly marked “to be implemented,” and `Simulator.step()` updates only trains and time. There is no ongoing passenger arrival process or passenger route execution in this flow. The README lists passenger boarding, multi-line passenger trips, and a graph representation for Dijkstra as TODOs. Project goals such as metrics collection and optimization in `.claude/CLAUDE.md` are intentions, not capabilities established by this startup and step flow.
