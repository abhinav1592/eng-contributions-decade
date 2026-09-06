# 03 — Hardware types and interface families (mental model)

**Artifact:** `../03-hardware-types-interfaces.html`
**Evidence:** pinned clone `.src-pinned/ironic` @ `26c022f1353e40f36456905620e8c1aca8cf1a31`

## The one-paragraph model

Ironic has no such thing as "a driver for a server". It has **thirteen interface families** —
power, management, boot, deploy, network, storage, inspect, raid, bios, firmware, console,
rescue, vendor — and for each family there are several interchangeable implementations
registered as setuptools entry points in `pyproject.toml` (`:48`-`:138`). A **hardware type**
is not a driver; it is a *bundle* that says, for each family, which implementations are
allowed and in what preference order. Every node picks exactly one implementation per family,
and those thirteen choices are stored as thirteen columns on the node row
(`ironic/objects/node.py:165`-`:177`).

## How a node actually resolves its interfaces

1. You set `driver=redfish` on the node. That string is just stored (`node.driver`).
2. The conductor looks `redfish` up in the `ironic.hardware.types` entry points
   (`pyproject.toml:140`-`:146`) and instantiates the class — here
   `RedfishHardware` (`ironic/drivers/redfish.py:32`).
3. That class exposes `supported_<family>_interfaces` properties, defined by the abstract base
   `AbstractHardwareType` (`ironic/drivers/hardware_type.py:30`) and given sane defaults by
   `GenericHardware` (`ironic/drivers/generic.py:40`). The lists are **ordered by preference** —
   e.g. `supported_deploy_interfaces` returns autodetect, direct, ansible, ramdisk, anaconda,
   bootc, custom-agent in that order (`ironic/drivers/generic.py:52`).
4. The operator's `ironic.conf` narrows that with `enabled_<family>_interfaces`
   (`ironic/conf/default.py:137`; the loader that wires every family is
   `ironic/common/driver_factory.py:566`).
5. `default_interface()` (`ironic/common/driver_factory.py:118`) walks the supported list and
   takes the first entry that is also enabled. That name is written into
   `node.power_interface`, `node.deploy_interface`, and so on.

So "which driver runs" is the intersection of *what the hardware type allows* and *what the
operator turned on*, resolved once, per family, per node.

## The split that actually matters: out-of-band vs in-band

- **Out-of-band** work goes to the server's BMC over Redfish (HTTPS 443) or IPMI (UDP 623).
  It works with the server powered off and with nothing running on the host. This is power
  (`pyproject.toml:108`), management (`:94`), bios (`:48`), firmware (`:82`), console (`:64`)
  and vendor (`:133`).
- **In-band** work needs the node to have booted the **IPA ramdisk** (ironic-python-agent) and
  to be heartbeating back to the conductor. This is deploy (`:71`) and rescue (`:122`).
- Three families have a driver on *each* side of that fence, and choosing the wrong one is a
  classic beginner mistake: `inspect` (`agent` in-band vs `redfish`/`idrac-redfish`
  out-of-band, `:87`), `raid` (`agent` vs `redfish`, `:115`), and `boot` (`pxe`/`ipxe`/`http`
  over the provisioning network vs `redfish-virtual-media`, which is pure BMC, `:54`).
- `network` (`:102`) and `storage` (`:127`) are on neither side — they call Neutron and Cinder.

## Confusions worth naming early

- **There is no `iscsi` deploy interface.** It was removed upstream. The complete list is
  `anaconda, ansible, autodetect, bootc, custom-agent, direct, fake, noop, ramdisk`
  (`pyproject.toml:71`-`:80`). Old blog posts and old docs still mention `iscsi`; they are stale.
- **`direct` is the normal one.** Its entry point maps to `AgentDeploy`
  (`pyproject.toml:77`) — "direct" refers to the node pulling the image directly, not to
  bypassing the agent.
- **Hardware types inherit from each other.** `intel-ipmi` subclasses `ipmi`
  (`ironic/drivers/intel_ipmi.py:17`) and only overrides management; `idrac` subclasses
  `redfish` (`ironic/drivers/drac.py:36`) and swaps in Dell variants. Reading the parent first
  saves a lot of confusion.
- **`manual-management` is a real, supported type** (`ironic/drivers/generic.py:103`), for
  hardware with no usable BMC. Its power interface is `agent_power`/`fake` and its management
  is `noop` — a human is expected to press the button.
- **`enabled_*` is an allow-list, not a selector.** Putting one driver in
  `enabled_deploy_interfaces` makes it the effective default; putting five in means the order
  in the hardware type's `supported_*` list decides.
- **A node's interface columns are sticky.** Once resolved and written to the node row, they do
  not silently change when you edit `ironic.conf`.

## Where to look next in the source

- `pyproject.toml:48`-`:146` — every interface family and every hardware type, in one screen.
- `ironic/drivers/generic.py` — the defaults nearly every hardware type inherits.
- `ironic/common/driver_factory.py:118` — the actual "which driver wins" algorithm.
- `ironic/objects/node.py:165`-`:177` — the thirteen columns the answer is stored in.
