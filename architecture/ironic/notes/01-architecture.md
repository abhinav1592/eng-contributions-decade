# 01 — System architecture

## Mental model

`ironic-api` is a stateless REST front end. It validates your request, writes your *intent*
into the database, and hands the actual work to a conductor over RPC. **The conductor is the
only process that ever touches hardware** — everything else is bookkeeping. Every node is
owned by exactly one live conductor, chosen by a hash ring over `node.uuid`. Nova, Glance,
Neutron and Keystone are all optional: Ironic standalone is a complete provisioning service,
which is exactly what Bifrost and Metal3 drive.

## The four layers

1. **Control plane** — `ironic-api` plus the two pieces of shared state, the database and
   the message bus. GET requests are answered straight from the DB
   (`ironic/api/controllers/v1/node.py:2335`); anything that changes hardware becomes an RPC
   to the owning conductor (`ironic/api/controllers/v1/node.py:445`).
2. **Conductor layer** — one or more `ironic-conductor` processes, each in a
   `conductor_group` (`ironic/conf/conductor.py:279`), heartbeating every 10s
   (`ironic/conf/conductor.py:42`). The hash ring (`ironic/common/hash_ring.py:32`, built on
   `tooz.hashring`) maps `node.uuid` → one live conductor, and the API turns that into the
   RPC topic `ironic.conductor.<host>` (`ironic/conductor/rpcapi.py:267`).
3. **Hardware interface layer** — the node's hardware type resolves into concrete interface
   drivers: power/management/bios/raid (out-of-band), deploy/inspect/clean/rescue (in-band,
   via IPA), and the network interface (`pyproject.toml`, the
   `ironic.hardware.interfaces.*` entry points).
4. **Physical hardware** — the BMC (Redfish 443 / IPMI 623), the PXE/iPXE + HTTP boot path,
   and the machine itself running the ironic-python-agent ramdisk.

## What first-time readers get wrong

- **`power_state` vs `provision_state`** — two independent columns on the same row
  (`ironic/objects/node.py:133` and `:140`). "Is it switched on?" versus "where is it in its
  life?". A node can be `power on` and `available`, sitting idle running nothing.
- **`reservation` is not a state.** It is a nullable string column holding the hostname of
  the conductor currently holding the node's lock (`ironic/objects/node.py:125`).
  `target_provision_state` (`:142`) is the "headed toward" companion to `provision_state`.
- **Out-of-band vs in-band.** Out-of-band = conductor → BMC, works with the machine powered
  off. In-band = conductor → IPA ramdisk running *on* the machine, so it needs the machine
  booted into the agent first. This is why a deploy always opens with a power-cycle-and-PXE
  dance.
- **Standalone vs full OpenStack.** Nothing below the API line changes between them. Only
  who is calling, and whether Keystone is in the path.

## Routes worth tracing in the viewer

1. **The happy path** — follow only the bright (emphasis) arrows: Nova → ironic-api → bus →
   conductor → in-band interfaces → node. That is a deploy, end to end.
2. **The ownership path** — `conductor → DB (reservation)` plus `hash ring → conductor`.
   That pair *is* the HA story: kill a conductor, its heartbeat stops, the ring rebuilds
   without it, another conductor takes over its nodes.
3. **The two hardware paths** — `out-of-band → BMC → node` versus `in-band → node`. Same
   machine, completely different networks.

All line references verified against `openstack/ironic` @ `26c022f1353e40f36456905620e8c1aca8cf1a31`.
