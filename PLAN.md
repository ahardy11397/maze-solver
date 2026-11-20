# PLAN — maze-solver: a Memelang graduate-level demo

Build a maze/pathfinding CLI entirely in Memelang. Beta demonstration
program: the algorithms-heavy assignment (BFS, DFS, grid representation).

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits.
- 100% Memelang. Grids are `List<List<Int>>` (0 open, 1 wall).
- JSON persistence in `data/` via Optional record readers/writers; missing/
  corrupt file = clean Optional handling, never a traceback.
- Everything must parse with `--provider heuristic`.
- Test suites run in the FOREGROUND only.

## Domain and CLI (thin argv-router main + logic modules; unknown command →
usage + exit 1; every error = one stderr line + exit 1)

    maze load <file>            # maze file: rows of 0/1 chars, e.g. "S..#"
                                # S = start, E = end, # = wall, . = open;
                                # rectangular, all rows same length
    maze solve-bfs <file>       # shortest path length + rendered path
    maze solve-dfs <file>       # any path (lexicographic neighbor order:
                                # up, down, left, right), rendered
    maze render <file>          # print the maze
    maze stats <file>           # open cells, wall cells, dimensions

Output rules: solve prints the maze with the path marked `*`, then one
summary line `path length <n>` (number of steps). No path: print
`no path exists` (exit 0 — that is a valid answer, not an error). Invalid
maze (no S/E, ragged rows, bad chars) = one stderr line + exit 1.
BFS must be a real queue (List used as index-walked queue, no recursion);
DFS is recursive with a visited `Set<String>` of "r,c" keys.

## Tests — tests/meme/*.meme, `meme test`

At minimum: BFS shortest length on a fixed maze (hand-computed); BFS vs
wall-detour case where greedy path would fail; no-path maze (walled-off E);
DFS finds A path (assert same start/end and validity: every step moves to
an adjacent open cell) even if not shortest; cycle safety (DFS terminates
on a maze with loops); ragged-row rejection; missing-S rejection; corrupt
data tolerance via Optional readers. Fixed mazes, unique scratch files,
cleaned up.

## Acceptance ("done means")

1. `meme test` python — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green.
3. `meme build main.meme -o maze_bin` — end-to-end demo: solve a 10x10
   maze with BFS and DFS, show both renderings, show a no-path case.
   Transcript in your final summary.
4. README.md: maze file format, algorithms, CLI usage, tests.
5. Final message: files, test count per target, demo transcript,
   deviations with reasons.
