# 02 — The node provision state machine (mental model)

**Artifact:** `../02-node-state-machine.html`
**Evidence:** pinned clone `.src-pinned/ironic` @ `26c022f1353e40f36456905620e8c1aca8cf1a31`

## The one-paragraph model

A bare metal node is a *row in a database*, and that row has a column called
`provision_state` that says where the machine is in its life. You never set that column
directly. You `PUT` a **verb** at the API — `manage`, `provide`, `deploy`, `delete` — and a
conductor moves the node through a fixed graph of states until it reaches a resting place.
The whole graph is built by hand, one line at a time, in
`ironic/common/state_machine.py` (`machine = fsm.FSM()` at `:59`). There are only six
resting ("stable") states: `enroll`, `manageable`, `available`, `active`, `error`, `rescue`
(`ironic/common/states.py:254`). Everything else is a state the node is passing *through*,
and something — usually the conductor — is expected to move it along.

## The happy rail, with who does what

1. You create the node. It lands in `enroll`. Nothing has been checked yet.
2. You `PUT target=manage`. That is verb #1 (`ironic/common/states.py:24`). The node goes to
   `verifying` (`state_machine.py:287`) and the **conductor** calls the BMC to test the
   credentials. Success → `manageable` (`:290`); failure → straight back to `enroll` (`:293`),
   not forward. That backwards failure is the first thing that surprises people.
3. You `PUT target=provide`. This does **not** go to `available` directly — it goes to
   `cleaning` (`:184`). The conductor wipes the disks, and only when cleaning reports `done`
   does the node become `available` (`:157`).
4. You `PUT target=active` with an image. Node → `deploying` (`:86`).
5. The conductor writes the image, then usually parks the node in `wait call-back` (`:114`)
   until the in-band IPA ramdisk heartbeats back; `resume` (`:117`) picks it up again.
6. `done` (`:132`) → `active`. Finished.

Only steps 2, 3 and 4 are things *you* send. `wait`, `resume`, `done` and `fail` are
internal events the conductor fires — they are not in `VERBS` and you cannot PUT them.

## Confusions worth naming early

- **`provision_state` is not `power_state`.** They are two different columns on the node row
  (`ironic/objects/node.py:140` and `:133`). A node can be `available` and powered on, or
  `active` and powered off. Power is synchronised by a background thread; provision state is
  driven by verbs.
- **`target_provision_state` is a memory of intent** (`ironic/objects/node.py:142`). When the
  node is `deploying`, its target is `active`; when it is `cleaning` on the way in, its target
  is `available`. That is how the conductor knows where to continue after a restart.
- **`reservation` is not a state.** It is a nullable string column holding the *hostname of the
  conductor currently holding the lock* (`ironic/objects/node.py:125`). A node stuck with a
  reservation is a locking problem, not a lifecycle problem.
- **The state string is literally `wait call-back`** — with a space and a hyphen
  (`ironic/common/states.py:90`). Do not go looking for `deploy_wait`.
- **`error` is rarer than you think.** Most failures land in a `<verb> failed` state
  (`deploy failed`, `clean failed`, `inspect failed`). The bare `error` state is reached when
  *deleting* itself fails (`state_machine.py:151`), and it can be recovered with `rebuild`
  (`:196`) or `delete` (`:199`).
- **`cleaning` sits on both edges of the life.** `provide` cleans on the way in (`:184`) and
  `delete` cleans on the way out (`ACTIVE -> DELETING` `:140`, `DELETING -> CLEANING` `:154`).
  A node basically never becomes `available` without passing through `cleaning`.
- **`inspect` only runs from `manageable`** (`:203`) and returns to `manageable` (`:206`).
  It is not something you can do to a deployed node.

## What the diagram deliberately leaves out

The FSM has 30-plus states; the lifecycle canvas has three bands and fifteen usable slots, so
the drawing is curated, not complete. Specifically:

- `verifying` is drawn as part of the `manage` arrow rather than as its own box.
- `clean wait`, `inspect wait` and the three rescue states are folded into their parent boxes
  (their sublabels name them).
- `adopting` / `adopt failed` (`state_machine.py:296`-`:309`), `deploy failed`, the three
  `hold` states, and the whole `service*` family (`:312`-`:373`, verb `service` at `:321`)
  are not drawn at all.
- `DELETING -> CLEANING` (`:154`) is stated in the `Deleting` box's sublabel rather than drawn,
  because that one arrow could not be routed without crossing another.

One rendering caveat: the archify lifecycle renderer always paints a continuous green "phase
rail" behind the top row. Read the **labelled** arrows as the real transitions — there is no
direct `manageable -> available` edge in the FSM.

## Where to look next in the source

- `ironic/common/state_machine.py` — the whole graph, readable top to bottom.
- `ironic/common/states.py:24` — `VERBS`, i.e. every legal `{"target": ...}` value.
- `ironic/common/states.py:254`/`:257`/`:289` — stable, unstable and failure state sets.
