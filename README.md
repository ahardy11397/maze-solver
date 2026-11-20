# maze-solver: A Memelang Graduate-Level Demo

A complete maze/pathfinding CLI built **100% in Memelang** (`meme`), demonstrating graduate-level multi-module architecture, generic 2D-list grid representation (`List<List<Int>>`), non-recursive index-walked queue Breadth-First Search (BFS), recursive Depth-First Search (DFS) with visited generic `Set<String>` collections, null-safe `Optional<T>` JSON persistence in `data/`, and deterministic cross-compilation across all 6 Memelang compiler targets (**Python**, **Rust**, **C++20**, **Go**, **C11**, and **TypeScript**).

---

## 1. Maze File Format

Mazes are defined in plaintext files as rectangular ASCII grids where all rows have the identical length:

| Character | Meaning | Grid Representation (`List<List<Int>>`) |
|---|---|---|
| `S` | Start position | `0` (open cell) |
| `E` | Destination position | `0` (open cell) |
| `.` or `0` | Open passage | `0` (open cell) |
| `#` or `1` | Wall obstacle | `1` (wall cell) |

### Validation Rules
A maze is validated strictly by `maze_loader.meme`:
- **Rectangular:** Every row must have the exact same length (no ragged rows).
- **Start Cell:** Exactly one `S` cell must exist. Missing or multiple `S` cells are rejected.
- **End Cell:** Exactly one `E` cell must exist. Missing or multiple `E` cells are rejected.
- **Valid Characters:** Only `S`, `E`, `#`, `1`, `.`, and `0` are permitted. Any unexpected character triggers an error.
- **Non-Empty:** The file must contain at least one valid row.

On validation failure, the CLI writes exactly one descriptive error line to `stderr` and terminates with status code `1`.

---

## 2. Algorithms

The solver implements two classic pathfinding algorithms in pure Memelang:

### Breadth-First Search (`solve-bfs`)
- **Queue Implementation:** Implemented as a genuine FIFO queue using an index-walked `List<QueueNode>` (no recursion). Each queue node stores `(r, c, parent_idx, dist)`.
- **Shortest Path Guarantee:** Guaranteed to find the minimum step distance from `S` to `E`.
- **Neighbor Exploration Order:** Lexicographical order:
  1. **Up:** `(r - 1, c)`
  2. **Down:** `(r + 1, c)`
  3. **Left:** `(r, c - 1)`
  4. **Right:** `(r, c + 1)`
- **Backtracking:** Once destination `E` is dequeued, the shortest path is reconstructed by walking `parent_idx` backward to the root and reversing the result.

### Depth-First Search (`solve-dfs`)
- **Recursive Traversal:** Implemented recursively with a `Set<String>` of `"r,c"` visited coordinates.
- **Neighbor Order:** Explores in deterministic lexicographic order: Up, Down, Left, Right.
- **Cycle Safety:** Guarantees termination even in open rooms or mazes with extensive cycles/loops by maintaining visited coordinate keys.
- **Path Validity:** Finds an admissible path where every consecutive step moves to an orthogonally adjacent open cell (`Manhattan distance == 1`), even when the path is not the shortest.

---

## 3. Architecture & Modules

The application is decomposed into clean, single-responsibility modules:

- **`models.meme`**: Defines core domain records:
  - `MazeRecord`: Stored dimensions, start/end coordinates, and serialized grid text.
  - `QueueNode`: BFS queue entry with coordinates, parent pointer index, and cumulative distance.
  - `ValidationResult`: Boolean validity status and failure message.
- **`maze_loader.meme`**:
  - File reading and row normalization (`clean_maze_lines`).
  - Comprehensive structural validation (`validate_maze_lines`).
  - 2D grid synthesis (`build_grid_from_data` -> `List<List<Int>>`).
  - Strongly-typed JSON serialization and deserialization in `data/` using `Optional<MazeRecord>`.
- **`bfs_solver.meme`**: Queue-based BFS algorithm returning ordered step coordinates.
- **`dfs_solver.meme`**: Recursive DFS algorithm with `Set<String>` visited tracking.
- **`maze_renderer.meme`**: Renders plaintext mazes and renders solved paths with intermediate steps marked with `*` while preserving `S`, `E`, and `#`.
- **`main.meme`**: Thin command-line argument router (`argument count`, `argument <n>`) with single-line `stderr` error reporting.

---

## 4. CLI Usage

### Build Native Executable

Build the standalone native binary via Memelang's C11 compiler backend:

```bash
uv run --project /mnt/ssd/projects/meme meme build main.meme -o maze_bin
```

### Commands

#### 1. `maze load <file>`
Validates `<file>` and persists it as a JSON record in `data/maze.json`:
```bash
./maze_bin load mazes/maze_10x10.txt
# Output: loaded maze 10x10
```

#### 2. `maze solve-bfs <file>`
Finds the shortest path using BFS, renders the maze with the path marked as `*`, and prints the path step count:
```bash
./maze_bin solve-bfs mazes/maze_10x10.txt
# Or solve from persisted JSON state:
./maze_bin solve-bfs data/maze.json
```
Output:
```text
S.....#...
*####.#.#.
*#....#.#.
*#.####.#.
***#....#.
##*#.####.
***#.#****
*###.#*##*
*****#*.#*
####***#.E
path length 28
```

#### 3. `maze solve-dfs <file>`
Finds a path using recursive DFS, renders the path with `*`, and prints the step count:
```bash
./maze_bin solve-dfs mazes/maze_10x10.txt
```

#### 4. `maze render <file>`
Prints the raw maze grid:
```bash
./maze_bin render mazes/maze_10x10.txt
```

#### 5. `maze stats <file>`
Prints grid dimensions, count of open cells, and count of wall cells:
```bash
./maze_bin stats mazes/maze_10x10.txt
# Output:
# dimensions: 10x10
# open cells: 60
# wall cells: 40
```

#### 6. No-Path Handling
When a maze has no reachable path from `S` to `E`, `solve-bfs` and `solve-dfs` print `no path exists` and exit cleanly with status code `0`:
```bash
./maze_bin solve-bfs mazes/no_path_10x10.txt
# Output:
# no path exists
# Exit code: 0
```

#### 7. Error Handling Contract
- Unknown commands print usage and exit with code `1`.
- Missing or invalid arguments print usage and exit with code `1`.
- Missing or malformed maze files write a single error line to `/dev/stderr` and exit with code `1`.

---

## 5. JSON Persistence & Fault Tolerance

Mazes are persisted in `data/` using Memelang's native JSON record serialization:
- **Writer:** `write rec as json to file "data/maze.json"`
- **Reader:** `read json record MazeRecord from file path or else nothing`

Missing or corrupt JSON files are handled via clean `Optional<MazeRecord>` evaluation:
- If a file is missing or corrupted, the reader evaluates safely to `nothing`.
- The CLI detects `nothing`, writes `Error: missing or corrupt maze file: <path>` to `stderr`, and exits with code `1`.
- Tracebacks, panics, and segmentation faults are strictly prevented.

---

## 6. Test Suite

The test suite covers 19 unit and integration tests across 4 test modules in `tests/meme/`:

1. **`01_bfs_test.meme`** (3 cases):
   - `bfs_fixed_maze_shortest_path`: Hand-computed 12-step path on a 5x5 obstacle maze.
   - `bfs_vs_greedy_wall_detour`: Verifies BFS escapes a greedy trap dead-end.
   - `bfs_no_path_walled_off`: Verifies empty path returned when destination is blocked.
2. **`02_dfs_test.meme`** (3 cases):
   - `dfs_finds_valid_path_and_not_shortest`: Verifies DFS finds an 8-step path where BFS finds a 4-step path; verifies every step is adjacent and open.
   - `dfs_cycle_safety_loops`: Verifies termination and valid path on an open 4x4 room with loops.
   - `dfs_no_path_walled_off`: Verifies empty path on blocked destination.
3. **`03_validation_test.meme`** (8 cases):
   - Rejection of ragged rows, missing start `S`, missing end `E`, multiple `S`, multiple `E`, invalid characters, and empty files.
4. **`04_persistence_test.meme`** (5 cases):
   - JSON record round-trip persistence in scratch files.
   - Corrupt JSON tolerance returning `nothing`.
   - Missing file tolerance returning `nothing`.
   - Cell count statistics (open vs wall).
   - Text rendering with path `*` overlay.

### Running the Tests Across All 6 Targets

Run all tests in the foreground across every supported backend:

```bash
# 1. Python 3 target
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target python

# 2. Rust target (standalone rustc)
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target rust

# 3. C++20 target
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target cpp

# 4. Go target (standalone go toolchain)
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target go

# 5. C11 target (gcc -std=c11)
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target c

# 6. TypeScript target (tsc + node)
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target ts
```
All six targets pass with `0 failed`.
