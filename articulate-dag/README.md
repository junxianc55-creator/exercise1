# Articulate DAG project folder

This folder includes the four graph functions, their tests, synthetic example data, and all four previously extracted Chabot-to-CSU East Bay major agreements. Original articulation mappings are preserved; the major files now include separately sourced Chabot prerequisite records.

## Start in VS Code

1. Extract this ZIP and open the `articulate-dag` folder in VS Code.
2. Open Terminal > New Terminal.
3. Run `npm test`. No dependency installation is needed for the tests.

## Folder contents

```text
articulate-dag/
├── graph.js
├── graph.test.js
├── courses.example.json
├── package.json
├── README.md
└── data/
    ├── computer_science.json
    ├── computer_engineering.json
    ├── biochemistry.json
    └── physics.json
```

| File | Major | Agreement year | JSON schema |
|---|---|---|---|
| `data/computer_science.json` | Computer Science | 2026-2027 | 1.0.0 |
| `data/computer_engineering.json` | Computer Engineering | 2026-2027 | 1.0.0 |
| `data/biochemistry.json` | Biochemistry — Chemistry Education concentration | 2025-2026 | 1.1.0 |
| `data/physics.json` | Physics | 2026-2027 | 1.2.0 |

## What is ready, and what remains

The graph algorithms and tests are ready to run. The four major files contain articulation mappings and requirement groups, including AND/OR bundles and elective thresholds. Their `prerequisite_data` now contains rich, sourced rules, including user-clarified entries and policy caveats. The files cannot be passed directly to `graph.js`.

For a real course schedule, review the user-clarified rules and remaining policy caveats, selected pathway and concurrent-enrollment constraints before adapting to the graph functions. Use `courses.example.json` for the included synthetic demo tests. The Biochemistry agreement is for 2025–2026; the other three are for 2026–2027.

This package contains the code and data created so far. A frontend (`index.html`, Cytoscape rendering, and the semester board) has not yet been built.

---

# Articulate DAG: graph.js

Four browser-compatible ES module exports, with no runtime dependencies:

| Function | Returns |
|---|---|
| `topoOrder(courses)` | Course IDs in Kahn topological order |
| `criticalPath(courses, completed = [])` | `{ path, semesters, semesterByCourse }` |
| `availableNow(courses, completed = [])` | Available, unfinished course IDs |
| `validatePlan(courses, plan)` | `{ valid, violations, unscheduled }` |

## Run the tests

Open this folder in VS Code and run in its terminal:

```bash
npm test
```

Node.js 18 or newer is required for the test runner. No `npm install` is needed. The 14 tests passed on Node.js v24.19.0. Only the tests use Node APIs; `graph.js` can be imported directly into a browser module.

## Input contract

```js
const courses = [
  { id: 'A', prereqs: [] },
  { id: 'B', prereqs: ['A'] },
  { id: 'C', prereqs: ['A'] },
  { id: 'D', prereqs: ['B', 'C'] }
];
```

This is a synthetic example, not verified Chabot catalog data. Optional metadata such as `title` and `units` is allowed and is not modified.

- `id` must be unique. Every prerequisite must refer to an included course.
- Every course must provide `prereqs`. Use `[]` only when no prerequisites is verified. Missing or null data throws an error.
- `prereqs` means AND: every listed prerequisite is required.
- These algorithms operate on selected courses. Resolve elective/OR options into an explicit selected pathway before calling them; the module deliberately rejects expression objects rather than flattening OR into AND.
- `completed` accepts an array or Set of known IDs. Completion is accepted as an input fact; the user need not separately mark all ancestors of an already-completed course.
- Each unfinished course takes one semester; prerequisites must finish in an earlier semester. The semester result is a lower bound that assumes unlimited parallel enrollment, availability every semester, and no unit cap. Corequisites, grades, placement rules, and equivalent-course credit decisions are outside this module.
- Do not pass an ASSIST articulation graph as a prerequisite graph. The four extracted agreement files do not contain `prereqs`. Use the new sourced registry as research input; resolve its review items and richer constraints before adapting it for real scheduling. Articulation AND/OR rules and elective thresholds remain separate from these prerequisite algorithms.

## Four small examples

```js
import {
  topoOrder, criticalPath, availableNow, validatePlan
} from './graph.js';

const courses = [
  { id: 'A', prereqs: [] },
  { id: 'B', prereqs: ['A'] },
  { id: 'C', prereqs: ['A'] },
  { id: 'D', prereqs: ['B', 'C'] }
];

topoOrder(courses);
// ['A', 'B', 'C', 'D']

criticalPath(courses, ['A']);
// {
//   path: ['B', 'D'],
//   semesters: 2,
//   semesterByCourse: { A: 0, B: 1, C: 1, D: 2 }
// }
// B and C run in parallel. C -> D is an equally long path;
// the function returns one deterministic critical path.

availableNow(courses, ['A']);
// ['B', 'C']

validatePlan(courses, [['A'], ['B', 'C'], ['D']]);
// { valid: true, violations: [], unscheduled: [] }

validatePlan(courses, [['A', 'B'], ['C', 'D']]);
// valid: false
// B requires A before semester 1.
// D requires C before semester 2.
```

Completed courses can be supplied to the plan checker:

```js
validatePlan(courses, {
  completed: ['A'],
  semesters: [['B', 'C'], ['D']]
});
// { valid: true, violations: [], unscheduled: [] }
```

Plan semesters are numbered from 1 in violation objects. Empty semesters are allowed. A valid partial plan can still have `unscheduled` courses; `valid` does not mean degree completion. Course IDs in `violations` can be used to mark semester-board cards red.

## Error handling

Invalid course data or a cycle throws an Error. A cycle error has `code === 'CYCLE_DETECTED'` and `blockedCourseIds`; that list may include descendants blocked by the cycle, not just cycle members. Invalid plan shape also throws. For a well-formed plan, unknown scheduled IDs, duplicates, already-completed courses, and unmet prerequisites are returned as violation objects.

All functions reject cyclic graphs. Kahn's algorithm and the course-graph computations take O(V + E) time. Plan validation additionally scans scheduled courses and their prerequisites. Inputs are never mutated.

## Files

- `graph.js`: the four functions and shared validation helpers.
- `graph.test.js`: 14 executable tests, including a small example for each function.
- `courses.example.json`: synthetic demo data.
- `package.json`: ES module and test-runner configuration.


## Prerequisite research update (2026-09-24)

See `PREREQUISITE_REVIEW.md` for all 35 Chabot courses and the screenshot clarifications. Each major JSON now embeds its course records in `prerequisite_data.records`; the shared `data/chabot_prerequisites.json` adds supporting courses and explicit semantics. These rich records are not direct input to the AND-only graph functions. Original articulation mappings and agreement years are preserved. Run `python3 check_prerequisites.py` after installing the version in `requirements-validation.txt` for structural checks.

Three previously ambiguous rules (CHEM 1A, BIOS 21B, BIOS 21C) now use the user-supplied screenshot groupings, labeled `user_clarified`. This is not independent official verification. CHEM 201 policy applicability remains open.
