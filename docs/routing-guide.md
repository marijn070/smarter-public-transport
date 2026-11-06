# Routing guide

## What is implemented

`src/routing/router.py` builds a static, weighted graph from a GTFS directory when a `TransitRouter` is created. It is independent of the simulation models in `src/core/line.py`, `src/core/station.py`, and `src/core/passenger.py`: those classes describe validated line, station, and passenger data, but the router reads GTFS files and does not accept those model instances.

The constructor inputs are:

- `gtfs_path`: directory containing `stops.txt`, `routes.txt`, `trips.txt`, and `stop_times.txt`.
- `line_names`: configured short names to include.
- `default_transfer_seconds`: transfer-edge weight, defaulting to 180 seconds.

The router keeps only routes whose `route_short_name` matches a configured name and whose `route_type` is one of `400` or `109` (the constant described in the source as U-Bahn and S-Bahn). For each included line and direction `0` and `1`, it selects the trip with the largest number of stop-time rows. It maps platform stops to `parent_station` when present, otherwise to the stop itself. Each chosen trip contributes directed edges between consecutive distinct parent stations, weighted by the next stop's arrival time minus the previous stop's departure time; non-positive weights are skipped. Times, including GTFS times beyond 24 hours, are parsed as seconds. Stations served by multiple included lines receive bidirectional transfer edges weighted by `default_transfer_seconds`.

The collected edges become a SciPy CSR sparse graph. Its nodes are `(parent_station_id, line_name)` pairs. Consequently, each line/direction is represented using the selected trip's ordered stop sequence and observed times, rather than a timetable search over every trip. This is the current implementation, not a claim about a complete or time-dependent journey planner.

## Query inputs and outputs

`find_route(origin_station_id, dest_station_id)` takes parent station IDs and returns a `RoutingResult` or `None`. It runs SciPy's directed Dijkstra query with all graph nodes at the origin as sources, then chooses the least-cost reachable node at the destination.

A `RoutingResult` contains:

- `segments`: ordered `RouteSegment` values, each with station ID, station name, and line name. A transfer station can appear twice, once for each line node.
- `total_cost_seconds`: sum of the selected travel and transfer edge weights.
- `transfers`: count of line changes along the reconstructed node path.

The query returns `None` if there is no usable graph node for either endpoint or no path connects them. If origin and destination IDs are equal, it returns a zero-cost result with one segment even if that station is not in the graph. The segment uses the station name if known, otherwise the ID; its line is whichever member of the station's line set is yielded first, or an empty string when no line is known. `get_lines_at_station` returns sorted served line names, and `get_all_station_ids` returns the station IDs represented in the router's station-to-lines index.

## Relationship to core models

`Station` in `src/core/station.py` holds an ID, names, and validated coordinates. `Line` in `src/core/line.py` holds an ordered list of stations, speed, and color, with distance helpers. `Passenger` in `src/core/passenger.py` holds origin, destination, and optional current station IDs. None of these classes currently invokes `TransitRouter` or stores its route result, so their existence should not be read as evidence that simulation passengers are routed through this GTFS graph.

## Grounded next steps (proposals)

These are possible follow-ups from existing README TODOs, not implemented behavior:

1. **Support multi-line passenger journeys.** Implement the README TODO “passenger generation for multi-line trips” by using a route result to select journeys whose path includes transfers. The router already has transfer edges and returns a transfer count, but `Passenger` currently stores only origin and destination IDs.
2. **Connect route choice to boarding.** Address “passenger boarding logic” in the README by defining how a passenger follows the route's line sequence when boarding and changing lines. The current `Passenger` model has no planned route or boarding state, and the router only returns a path; it does not board passengers.
3. **Align the network TODO with the existing router.** The README says “make lines a graph to implement Dijkstra”. Reconcile that TODO with `src/routing/router.py`, which already builds a Dijkstra graph from GTFS, and clarify whether the intended graph should use the simulation `Line` and `Station` models.

## Sources

- [src/routing/router.py](../src/routing/router.py)
- [src/core/line.py](../src/core/line.py)
- [src/core/station.py](../src/core/station.py)
- [src/core/passenger.py](../src/core/passenger.py)
- [README.md](../README.md)
