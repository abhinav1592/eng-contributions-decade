# 06 — Enroll → verify → provide, in plain English

Diagram: `06-enroll-verify-provide.html` (spec `06-enroll-verify-provide.sequence.json`)

All citations are `path:line` inside the pinned clone
`.src-pinned/ironic` at revision `26c022f1353e40f36456905620e8c1aca8cf1a31`.

## The one-paragraph mental model

Onboarding a machine into Ironic is a **ladder of trust**. First you make a
database row that merely *claims* a server exists (`ENROLL`) — nothing has been
dialled yet. Then you paste in the BMC address and credentials. Then you ask
Ironic to prove those credentials actually work (`manage` → `VERIFYING`). Only
once the BMC answers does the node reach `MANAGEABLE`, which means "Ironic can
drive this box, but it is not in the pool yet". Optionally you scan the hardware
(`inspect`). Finally `provide` wipes the disks through **automated cleaning**
(`CLEANING` → `CLEANWAIT` → `AVAILABLE`), and `AVAILABLE` is the only state a
scheduler will place an instance onto.

## API verbs vs automatic transitions — the key distinction

The diagram deliberately splits these. Only three arrows in the whole flow come
from a human:

| You type | HTTP | RPC the API makes |
|---|---|---|
| `openstack baremetal node create` | `POST /v1/nodes` | none — the row is written directly |
| `... node manage` | `PUT /v1/nodes/{id}/states/provision` `target=manage` | `do_provisioning_action(manage)` |
| `... node inspect` | same endpoint, `target=inspect` | `inspect_hardware` |
| `... node provide` | same endpoint, `target=provide` | `do_provisioning_action(provide)` |

`manage`, `provide`, `abort`, `adopt`, `unhold` and `service` are the targets
routed to `do_provisioning_action` — see `PROVISION_ACTION_STATES`
(`ironic/api/controllers/v1/node.py:133`) and the dispatch in
`_do_provision_action` (`node.py:1131`). `inspect` is special-cased to its own
RPC one branch earlier (`node.py:1106`).

**Everything else on the diagram is the conductor moving the state machine by
itself.** `ENROLL → VERIFYING`, `VERIFYING → MANAGEABLE`, `MANAGEABLE → CLEANING`,
`CLEANING → CLEANWAIT`, `CLEANING → AVAILABLE` are all `task.process_event(...)`
calls inside conductor code, not API calls.

## Step 1 — `POST /v1/nodes` puts you in ENROLL, not AVAILABLE

`NodesController.post` (`ironic/api/controllers/v1/node.py:2994`) sets
`node['provision_state'] = api_utils.initial_node_provision_state()`
(`node.py:3042`). That helper (`ironic/api/controllers/v1/utils.py:1190`) returns
`AVAILABLE` for API microversions below 1.11 and `ENROLL` from 1.11 onwards. If
you ever see a brand-new node land straight in `AVAILABLE`, you are talking to
Ironic with a pre-1.11 microversion.

Note what the POST handler does *before* writing the row: it generates a UUID and
resolves `conductor_group`, because `get_topic_for` needs a UUID to place the node
on the hash ring (`node.py:3028-3037`). Enrolling a node with a driver no
conductor supports fails right here.

Setting `driver_info` afterwards is a plain `PATCH /v1/nodes/{id}`
(`node.py:3212`), which the API forwards as `rpcapi.update_node` (`node.py:3342`).
**Still nothing has contacted the BMC.**

## Step 2 — `manage` is the credential check

`ConductorManager.do_provisioning_action` (`ironic/conductor/manager.py:1458`) has
an explicit branch for it (`manager.py:1498-1503`): when `action == 'manage'` and
the node is in `ENROLL`, it fires `task.process_event('manage', ...)` with
`call_args=(verify.do_node_verify, task)`. The FSM edge is
`ENROLL → VERIFYING` on `manage` (`ironic/common/state_machine.py:287`).

`verify.do_node_verify` (`ironic/conductor/verify.py:29`) is short enough to read
in one sitting, and it is the whole point of the state:

1. `task.driver.power.validate(task)` (`verify.py:37`) — for Redfish this is
   `redfish_utils.parse_driver_info(task.node)`
   (`ironic/drivers/modules/redfish/power.py:109-116`): are the required
   `driver_info` fields present and well-formed?
2. `task.driver.power.get_power_state(task)` (`verify.py:46`) — this is the part
   that **actually dials the BMC**. Redfish does `redfish_utils.get_system(...)`
   and maps `system.power_state` (`redfish/power.py:118-129`); IPMI shells out via
   `_power_status` (`ironic/drivers/modules/ipmitool.py:1080-1092`).
3. Any enabled **verify steps** are executed (`verify.py:54-62`).
4. On success it caches driver data (`verify.py:76`) and fires
   `task.process_event('done')` (`verify.py:81` / `verify.py:86`) →
   `VERIFYING → MANAGEABLE` (`state_machine.py:290`).
5. On any error it records node history and fires `task.process_event('fail')`
   (`verify.py:96`) → `VERIFYING → ENROLL` (`state_machine.py:293`).

That last edge is the one to remember: **a failed verify sends the node all the
way back to ENROLL**, not to a `VERIFYFAIL` state. Fix `driver_info` and run
`manage` again.

## Step 3 — `inspect` is optional and orthogonal

`MANAGEABLE → INSPECTING` on `inspect` (`state_machine.py:203`), and inspection
returns to `MANAGEABLE` (`state_machine.py:215` for the async
`INSPECTWAIT → MANAGEABLE` edge). The conductor side is
`inspection.inspect_hardware` (`ironic/conductor/inspection.py:29`): it wipes any
stale agent token/URL, calls `task.driver.inspect.inspect_hardware(task)`, and
then fires `done` or `wait` depending on whether the interface is out-of-band or
in-band. In-band inspection boots IPA and the ramdisk posts its inventory to
`POST /v1/continue_inspection`
(`ironic/api/controllers/v1/ramdisk.py:357`).

Inspection is **not** part of `provide`. It is a separate verb you may run zero or
many times while the node sits in `MANAGEABLE`.

## Step 4 — `provide` means "clean it, then publish it"

Back in `do_provisioning_action`, the `provide` branch (`manager.py:1476-1496`)
refuses maintenance and retired nodes, collects the automated clean steps, and
fires `task.process_event('provide', ..., call_args=(cleaning.do_node_clean, ...))`.
The FSM edge is `MANAGEABLE → CLEANING` on `provide`
(`ironic/common/state_machine.py:184`) — note the target is **CLEANING**, not
AVAILABLE. `provide` does not mean "mark available"; it means "start cleaning".

`cleaning.do_node_clean` (`ironic/conductor/cleaning.py:78`):

* If automated cleaning is disabled it short-circuits with `process_event('done')`
  and the node goes straight to `AVAILABLE` (`cleaning.py:99-112`).
* Otherwise it validates power and network (`cleaning.py:137-139`), then calls
  `task.driver.deploy.prepare_cleaning(task)` (`cleaning.py:173`), which for
  agent-based drivers boots the IPA ramdisk
  (`ironic/drivers/modules/agent_base.py:806`) and returns `CLEANWAIT`.
* Seeing `CLEANWAIT` it fires `process_event('wait')` (`cleaning.py:190-199`) and
  the worker exits.

Then the same heartbeat machinery as a deploy takes over: IPA POSTs
`/v1/heartbeat/{node}`, which reaches `HeartbeatMixin._heartbeat_clean_wait`
(`agent_base.py:558`). On the first heartbeat it calls `task.resume_cleaning()`
(`agent_base.py:570`), caches the agent's clean steps, sets them on the node
(`agent_base.py:576`) and calls `cleaning.continue_node_clean(task)`
(`agent_base.py:578`). Typical in-band steps are `erase_devices` and
`erase_devices_metadata`, whose priorities are read from config in
`agent_base.py:879-881`.

When the step list runs out, cleaning fires `done` and the FSM edge
`CLEANING → AVAILABLE` (`state_machine.py:157`) finally publishes the node.

## Common confusions, collected

* **`manage` does not "manage" anything.** It is a credential smoke-test. The
  state is literally called `VERIFYING`.
* **A failed verify lands in `ENROLL`, not a fail state** (`verify.py:96`,
  `state_machine.py:293`).
* **`provide` runs a disk wipe.** On real hardware `provide` can take a very long
  time; that is `erase_devices`, not a hang.
* **`MANAGEABLE` is not "ready".** Nova only schedules onto `AVAILABLE`.
* **The power interface and the BMC are different boxes on the diagram.** The
  power interface is Python code inside the conductor process; the BMC is the
  physical service processor it speaks Redfish or IPMI to.
