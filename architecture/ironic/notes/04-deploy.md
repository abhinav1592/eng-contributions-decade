# 04 — The deploy sequence, in plain English

Diagram: `04-deploy-sequence.html` (spec `04-deploy-sequence.sequence.json`)

All citations are `path:line` inside the pinned clone
`.src-pinned/ironic` at revision `26c022f1353e40f36456905620e8c1aca8cf1a31`.

## The one-paragraph mental model

Ironic deploys a physical server the same way you would if you did it by hand,
except a daemon does it. It phones the machine's **management controller (BMC)**
out-of-band to set a boot device and press the power button; the machine then
**network-boots a small Linux image called IPA** (ironic-python-agent) that lives
entirely in RAM; **IPA calls back to ironic-api**, gets told which node it is, and
starts heartbeating; the conductor then drives it through a list of **deploy
steps**, the biggest of which tells IPA to *download the disk image itself* and
write it to the local disk; finally the conductor installs a bootloader, points
the BMC at the disk, powers the agent off and the real instance on. Only then
does `provision_state` become `ACTIVE`.

## The five things that confuse first-time readers

### 1. `ironic-api` never touches the hardware.

`ironic-api` is a thin REST layer. `PUT /v1/nodes/{id}/states/provision` lands in
`_do_provision_action` (`ironic/api/controllers/v1/node.py:1073`), which picks a
conductor off the hash ring with `get_topic_for(rpc_node)` (`node.py:1077`) and
then makes **one RPC call** — `do_node_deploy` (`node.py:1085`). The HTTP request
returns `202 Accepted` immediately. Everything after that happens in
`ironic-conductor`.

The RPC method is `ConductorManager.do_node_deploy`
(`ironic/conductor/manager.py:917`). It takes an exclusive lock on the node,
validates, and hands off to `deployments.start_deploy`
(`ironic/conductor/deployments.py:139`), which spawns a background worker running
`deployments.do_node_deploy` (`deployments.py:207`).

### 2. There is no iSCSI deploy. The contrast is *how the image gets written*, not iSCSI vs direct.

The deploy interfaces that actually exist are listed in the entry points
(`pyproject.toml:71-80`):
`anaconda, ansible, autodetect, bootc, custom-agent, direct, fake, noop, ramdisk`.

* **`direct`** = `AgentDeploy` (`pyproject.toml:77` → `ironic/drivers/modules/agent.py`).
  This is the one the diagram draws. Its `write_image` deploy step
  (`agent.py:654`, priority 80) builds an `image_info` dict containing
  `instance_info['image_url']` plus a checksum, and then **hands that URL to IPA**
  (`agent.py:737`). The conductor does not stream the image; the agent pulls it.
* **`ramdisk`** = `RamdiskDeploy` (`ironic/drivers/modules/ramdisk.py:106`).
  It writes *nothing* to disk. It powers the node off, switches to the tenant
  network, calls `boot.prepare_instance`, and powers it back on — the machine
  runs a network-booted image forever.
* **`anaconda`** = `PXEAnacondaDeploy` (`ironic/drivers/modules/pxe.py:59`, deploy
  step at `pxe.py:71`). It boots the *distro's* installer over PXE with a
  kickstart file instead of using IPA to copy an image.

### 3. PXE vs virtual media is a *boot interface* choice, orthogonal to the deploy interface.

Boot interfaces (`pyproject.toml:54-62`): `pxe`, `ipxe`, `http`, `http-ipxe`,
`redfish-virtual-media`, `redfish-https`, `idrac-redfish-virtual-media`, `fake`.

* **Network boot** — `PXEBaseMixin.prepare_ramdisk`
  (`ironic/drivers/modules/pxe_base.py:138`) generates DHCP options
  (`pxe_utils.dhcp_options_for_instance`, `ironic/common/pxe_utils.py:500`), pushes
  them into the DHCP provider with `provider.update_dhcp` (`pxe_base.py:185`),
  writes a PXE/iPXE config file, caches kernel+ramdisk, and sets the boot device
  to PXE (`pxe_base.py:234`).
  The **iPXE chainload** is worth understanding: dnsmasq is tagged so that a
  *dumb firmware* PXE client is handed `ipxe.efi`/`undionly.kpxe` over TFTP, while
  a client that identifies itself as iPXE is instead handed the HTTP URL
  `CONF.deploy.http_url + /boot.ipxe` (`pxe_utils.py:562`, tag logic at
  `pxe_utils.py:565-580`). So the machine boots twice: firmware → iPXE over TFTP,
  then iPXE → kernel/initramfs over HTTP.
* **Virtual media** — `RedfishVirtualMediaBoot.prepare_ramdisk`
  (`ironic/drivers/modules/redfish/boot.py:1010`) skips DHCP entirely. It builds a
  deploy ISO, ejects and inserts it as a virtual CD over Redfish
  (`redfish/boot.py:1122`) and sets the boot device to `CDROM`
  (`redfish/boot.py:1129`). Note it can also *pre-inject* the agent token into the
  ISO (`redfish/boot.py:1053-1057`), which network boot cannot do.

In the diagram this is the dashed `alt boot: insert Redfish vmedia ISO` arrow.

### 4. The agent calls *back into the REST API*, not into the conductor.

This is the part that surprises everyone. IPA boots with no idea which node it is.

* **Lookup**: `GET /v1/lookup?addresses=<MACs>` →
  `LookupController.get_all` (`ironic/api/controllers/v1/ramdisk.py:130,143`).
  It finds the node by port MAC addresses, refuses if the provision state is not
  a state where lookup is allowed, and returns the node plus a freshly minted
  `agent_token`.
* **Heartbeat**: `POST /v1/heartbeat/{node_ident}` →
  `HeartbeatController.post` (`ramdisk.py:207,215`), which again resolves a topic
  and forwards over RPC with `rpcapi.heartbeat(...)` (`ramdisk.py:320`).
  Both endpoints are wired up in `ironic/api/controllers/v1/__init__.py:133-134`.
* On the conductor side that becomes `ConductorManager.heartbeat`
  (`ironic/conductor/manager.py:3554`) → `HeartbeatMixin.heartbeat`
  (`ironic/drivers/modules/agent_base.py:639`), which records `agent_url` and
  dispatches by provision state — `DEPLOYWAIT` goes to `_heartbeat_deploy_wait`
  (`agent_base.py:529`).

So the heartbeat is what *unblocks* the deploy. The conductor put the node into
`DEPLOYWAIT` and killed its worker; the heartbeat wakes a new one.

### 5. Conductor → agent is a plain HTTP command API, not a message bus.

Once the conductor knows `agent_url`, it POSTs to
`<agent_url>/<agent_api_version>/commands/` (`ironic/drivers/modules/agent_client.py:116`)
with an `agent_token` query parameter (`agent_client.py:200-241`). "JSON-RPC" in
Ironic means something *different* — it is an alternative to AMQP for
**api ↔ conductor** traffic. Do not confuse the two.

## The deploy steps, in priority order

`do_next_deploy_step` (`ironic/conductor/deployments.py:323`) walks the sorted step
list. For a `direct` deploy on an agent-driven driver the interesting ones are:

| prio | step | where |
|-----:|------|-------|
| 100 | `deploy` — reboots the node so IPA comes up, returns `DEPLOYWAIT` | `agent.py:353` |
| 80 | `write_image` — hands `image_url` + checksum + configdrive to IPA | `agent.py:654` |
| 60 | `prepare_instance_boot` — reads partition UUIDs back, installs the bootloader | `agent.py:743` → `configure_local_boot` `agent.py:830`, `client.install_bootloader` `agent.py:914`, `set_boot_to_disk` `agent.py:928`/`agent.py:214` |
| 40 | `tear_down_agent` — `sync`, then power the agent off (or reboot) | `agent.py:404` |
| 30 | `switch_to_tenant_network` | `agent_base.py:1162` |
| 20 | `boot_instance` — power the machine on for real | `agent_base.py:1184` |

When the last step returns, `do_next_deploy_step` fires `task.process_event('done')`
(`deployments.py:501`) and the node is `ACTIVE`.

A step that returns `states.DEPLOYWAIT` means "I am asynchronous" — the worker
exits and the next heartbeat (or an explicit `continue_node_deploy` RPC,
`ironic/conductor/manager.py:964`) resumes the list.

## Where the configdrive comes from

The configdrive is passed in on the original API call, stored by
`_store_configdrive` (`deployments.py:607`) called from `deployments.do_node_deploy` (`deployments.py:220`),
and later folded into the `image_info` payload sent to IPA as
`image_info['configdrive']` (`agent.py:725-729`) and into the step arguments
(`agent.py:734`). It is not a separate protocol —
it rides along with the write-image command.
