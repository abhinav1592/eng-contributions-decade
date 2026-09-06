# 05 — Cleaning, in plain English

Diagram: `05-cleaning.html` (spec `05-cleaning.workflow.json`)

All citations are `path:line` inside the pinned clone
`.src-pinned/ironic` at revision `26c022f1353e40f36456905620e8c1aca8cf1a31`.

## The one-paragraph mental model

A bare metal server is not a VM: when a tenant is finished with it, **the machine
still physically contains everything they left behind** — the bytes on the disks,
the RAID layout they configured, the BIOS settings they changed, possibly the
firmware they flashed. There is no hypervisor to throw away. So Ironic refuses to
hand a node to the next tenant until it has run a list of **clean steps** on it.
Cleaning is that list: the conductor asks every hardware interface "what clean
steps do you offer?", sorts them by priority, boots the IPA ramdisk if any of
them need to run *inside* the machine, and then executes them one at a time,
saving its place after each one so it can resume across a reboot.

## The five things that confuse first-time readers

### 1. Cleaning happens twice, and you only asked for it once.

There are two automatic entry points into `CLEANING`, plus one manual one:

- `provide` from `MANAGEABLE` — `MANAGEABLE --provide--> CLEANING`
  (`ironic/common/state_machine.py:184`). The API verb is handled at
  `ironic/conductor/manager.py:1476`, which calls `cleaning.do_node_clean`
  (`manager.py:1492`).
- **after tear-down** — when you delete an instance the node goes to `DELETING`,
  and when tear-down finishes it falls straight into cleaning:
  `DELETING --clean--> CLEANING` (`state_machine.py:154`).
- `clean` from `MANAGEABLE` with an explicit step list — `MANAGEABLE --clean-->
  CLEANING` (`state_machine.py:188`), RPC entry point
  `ConductorManager.do_node_clean` (`ironic/conductor/manager.py:1340`).

The first two are **automated cleaning**; the third is **manual cleaning**. The
code tells them apart by one line: `manual_clean = clean_steps is not None and
automated_with_steps is False` (`ironic/conductor/cleaning.py:92`), and later, in
the step loop, by looking at where the node is *heading*: `manual_clean =
node.target_provision_state == states.MANAGEABLE`
(`ironic/conductor/cleaning.py:242`).

That difference decides the ending. At the end of the run:

```python
event = 'manage' if manual_clean or node.retired else 'done'
```

(`ironic/conductor/cleaning.py:390`) — `done` means `CLEANING --> AVAILABLE`
(`state_machine.py:157`), `manage` means `CLEANING --> MANAGEABLE`
(`state_machine.py:189`). **Manual cleaning never makes a node schedulable.**

You can switch automated cleaning off, per node or globally:
`skip_automated_cleaning` (`ironic/conductor/utils.py:1227`) returns True when
`node.automated_clean` is False, or when it is unset and
`[conductor]/automated_clean` is off. The check is at
`ironic/conductor/cleaning.py:99`, and it simply fires `done` and moves on.

### 2. The step list is assembled from *every* interface, then sorted.

Nothing hard-codes "erase the disks". The conductor walks a dictionary of
interfaces and calls `get_clean_steps()` on each one
(`ironic/conductor/steps.py:174-181`). That dictionary is
`CLEANING_INTERFACE_PRIORITY` (`ironic/conductor/steps.py:30`):

```
vendor 7, power 6, management 5, firmware 4, deploy 3, bios 2, raid 1
```

Those numbers are **not** the step order — they are the tie-break. Every step
carries its own `priority` (set by the `@base.clean_step(priority=...)`
decorator, `ironic/drivers/base.py:2214`). The sort key is the pair:

```python
return (step.get('priority'),
        CLEANING_INTERFACE_PRIORITY[step.get('interface')])
```

(`ironic/conductor/steps.py:103-104`) sorted `reverse=True`
(`ironic/conductor/steps.py:134`). So: **highest step priority first; if two
steps tie, the interface higher in that table wins.** Steps with priority `0` are
dropped for automated cleaning (`ironic/conductor/steps.py:192`) — that is what
"disabled by default" means for a clean step.

For manual cleaning the sort is skipped entirely: your list is validated against
what the drivers actually offer (`_validate_user_clean_steps`, called from
`set_node_cleaning_steps`, `ironic/conductor/steps.py:323`) and then run **in the
order you wrote it**.

The classic in-band steps come from the IPA ramdisk, not from Ironic. Ironic only
overrides their priorities (`ironic/drivers/modules/agent_base.py:878-882`):

```python
new_priorities = {
    'erase_devices': CONF.deploy.erase_devices_priority,
    'erase_devices_metadata': CONF.deploy.erase_devices_metadata_priority,
}
```

with defaults documented at `ironic/conf/deploy.py:146` and `:153` — 10 and 99
respectively, "set to 0 and it will not run during cleaning".

### 3. In-band vs out-of-band decides whether a ramdisk boots at all.

Each clean step declares `requires_ramdisk` (default `True`) on its decorator
(`ironic/drivers/base.py:2217`). Out-of-band steps — a Redfish BIOS reset, a RAID
delete over the BMC — set `requires_ramdisk=False` (for example
`ironic/drivers/modules/redfish/bios.py:158`).

Before booting anything, the conductor tries
`_validate_clean_steps_early` (`ironic/conductor/cleaning.py:36`), and if
`all_steps_disable_ramdisk` says every requested step is out-of-band
(`ironic/conductor/cleaning.py:65`) it sets `disable_ramdisk = True` and
**skips the IPA boot entirely**. That is the fast path for "just reset the BIOS".

Otherwise `prepare_cleaning` runs. For the agent-based deploy interfaces that is
`prepare_inband_cleaning` (`ironic/drivers/modules/deploy_utils.py:743`), which
attaches the cleaning network (`deploy_utils.py:778`), reboots the node, and
returns `states.CLEANWAIT` (`deploy_utils.py:796`). The conductor sees that
return value and parks the node (`ironic/conductor/cleaning.py:190`).

### 4. `CLEANWAIT` is not an error — it is how async work is expressed.

A clean step is synchronous if it returns `None` and asynchronous if it returns
`states.CLEANWAIT` (`ironic/drivers/base.py:2236-2244`). When the step loop sees
`CLEANWAIT` it fires the `wait` event and *kills the worker thread*
(`ironic/conductor/cleaning.py:341-348`). Nothing is polling. The node sits in
`CLEANWAIT` (`ironic/common/states.py:138`) until something calls
`continue_node_clean` over RPC — normally the next agent heartbeat.

Because the loop can be re-entered at any time, it records its position before
every step: `node.clean_step = step` and `clean_step_index = step_index + ind`
(`ironic/conductor/cleaning.py:264-265`). That is what makes cleaning survive a
reboot in the middle of the list.

There is also a deliberate pause. A step literally named `hold` is intercepted
before it reaches any driver:

```python
if step_name == 'hold':
    task.process_event('hold')
    return EXIT_STEPS
```

(`ironic/conductor/steps.py:1075-1077`) — `CLEANING/CLEANWAIT --hold-->
CLEANHOLD` (`state_machine.py:168-169`). The node stays there until an operator
sends the `unhold` verb (`ironic/conductor/manager.py:1530-1536`), which is
`CLEANHOLD --unhold--> CLEANWAIT` (`state_machine.py:176`).

### 5. A failed clean puts the node in maintenance — on purpose.

`cleaning_error_handler` (`ironic/conductor/utils.py:544`) is the single place
failures land. If a clean step was actually executing it sets both:

```python
node.fault = faults.CLEAN_FAILURE
node.maintenance = True
```

(`ironic/conductor/utils.py:569-570`), tears the cleaning environment down, wipes
the agent token and URL, then fires `fail` → `CLEANFAIL`
(`ironic/common/states.py:145`). Maintenance mode is the point: a node whose
disks may still hold the previous tenant's data must not be schedulable. Getting
out is a deliberate operator act — `CLEANFAIL --manage--> MANAGEABLE`
(`state_machine.py:180`).

Abort is the same destination by a different road. `abort` is accepted from
`CLEANWAIT` and `CLEANHOLD` (`ironic/conductor/manager.py:1517-1527`); if the
current step is not `abortable` the conductor sets an `abort_after` flag and lets
the step finish first (`manager.py:1553-1571`, checked in
`continue_node_clean`, `ironic/conductor/cleaning.py:521`).

One more useful detail: on failure the conductor grabs the ramdisk's logs before
tearing it down — `driver_utils.collect_ramdisk_logs(task.node,
label='cleaning')` (`ironic/conductor/cleaning.py:334`), and again on success if
`[agent]/deploy_logs_collect = always` (`ironic/conductor/cleaning.py:362`).

## Things worth remembering

- Cleaning is about **the physical machine**, not about Ironic's bookkeeping.
  Turning it off is a real security decision, not a performance tweak.
- `priority` orders steps; the interface table only breaks ties.
- "Disabled" for a clean step means `priority == 0`, and it can still be invoked
  manually.
- `CLEANWAIT` means *a thread was deliberately released*, not *something is
  stuck*. If a node is wedged in `CLEANWAIT`, look for a missing heartbeat.
- `CLEANFAIL` always comes with `maintenance = True` when a step was running.
