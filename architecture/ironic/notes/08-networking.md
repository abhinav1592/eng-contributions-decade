# 08 — The three networking models, in plain English

Diagram: `08-networking-models.html` (spec `08-networking-models.architecture.json`)

All citations are `path:line` inside the pinned clone
`.src-pinned/ironic` at revision `26c022f1353e40f36456905620e8c1aca8cf1a31`.

## The one-paragraph mental model

A bare metal node has **at least three networks touching it, and they are not the
same network**: the *management network* your BMC answers on (Redfish/IPMI, alive
even when the machine is off), the *provisioning network* the node PXE-boots from
and where the IPA ramdisk phones home, and the *tenant network* the finished
instance is supposed to live on. Ironic's `network_interface` setting decides how
much of that plumbing Ironic itself does: `noop` does nothing, `flat` binds every
port to one shared network and leaves it there, and `neutron` creates a
provisioning port, binds it to a real switch port, and flips that switch port to
the tenant VLAN at the end of the deploy. Separately, `[dhcp]/dhcp_provider`
decides *who answers DHCP* — and that is a different setting.

## The five things that confuse first-time readers

### 1. `network_interface` is per node, not per cloud.

The three interfaces are registered entry points
(`pyproject.toml:102`, section `ironic.hardware.interfaces.network`) and every
node picks one. They all implement the same method names — `validate`,
`add_provisioning_network`, `remove_provisioning_network`,
`configure_tenant_networks`, `unconfigure_tenant_networks`,
`add_cleaning_network`, `add_inspection_network`, and so on — so the conductor's
code is identical in all three cases. What differs is what those methods *do*.

### 2. `noop` really does nothing. All of it.

`NoopNetwork` (`ironic/drivers/modules/network/noop.py:16`) is a class where
essentially every method body is `pass`:

```python
def add_provisioning_network(self, task):
    pass
```

(`ironic/drivers/modules/network/noop.py:81-86`; the same for
`configure_tenant_networks` at `:95-100` and `add_cleaning_network` at
`:109-114`). This is not a stub waiting to be written — it is a supported mode.
You use it when the switch is already configured by hand, when everything lives
on one lab VLAN, or when something outside Ironic (a standalone Bifrost-style
deployment) owns the network. Ironic will PXE-boot the node and never touch a
port anywhere.

### 3. `flat` is not "simple neutron" — it is "one network for both roles".

Look at what `flat` actually does. `add_provisioning_network` and
`configure_tenant_networks` call **the same private method**:

```python
def add_provisioning_network(self, task):
    self._bind_flat_ports(task)
...
def configure_tenant_networks(self, task):
    self._bind_flat_ports(task)
```

(`ironic/drivers/modules/network/flat.py:102-108` and `:117-122`, with
`_bind_flat_ports` at `flat.py:60`). That bind sets
`binding:host_id = node.uuid` and `binding:vnic_type = baremetal` on whatever VIF
is already attached (`flat.py:68-70`) — it does not move the port to a different
network, because there is only the one flat network.

The consequence is the important part: **there is no tenant isolation.** Every
node being provisioned and every node running a workload sits on the same L2
segment. `flat` still needs a Neutron network configured for cleaning, and warns
loudly at construction time if `[neutron]/cleaning_network` is unset
(`ironic/drivers/modules/network/flat.py:41-48`, config option at
`ironic/conf/neutron.py:41`).

### 4. `neutron` gives every node its own port — and then flips the VLAN.

`NeutronNetwork.add_provisioning_network`
(`ironic/drivers/modules/network/neutron.py:87-96`) creates real Neutron ports on
`[neutron]/provisioning_network` (`ironic/conf/neutron.py:50`) and records the
resulting VIF ids in `port.internal_info` (`neutron.py:62-74`). Cleaning,
rescuing, inspection and servicing each have their own network option
(`ironic/conf/neutron.py:41, 76, 111`).

Binding is what makes this more than paperwork. `plug_port_to_tenant_network`
(`ironic/drivers/modules/network/common.py:242`) builds:

```python
port_attrs = {'binding:vnic_type': neutron.VNIC_BAREMETAL,
              'binding:host_id': node.uuid}
binding_profile = {'local_link_information': local_link_info}
```

(`common.py:289-294`), where `local_link_information` is the node's
`port.local_link_connection` — the switch id and switch port name, exactly the
fields inspection's LLDP hook fills in. A Neutron ML2 driver reads that profile
and programs the physical switch port.

The flip itself is a **deploy step**:

```python
@base.deploy_step(priority=30)
def switch_to_tenant_network(self, task):
    with manager_utils.power_state_for_network_configuration(task):
        task.driver.network.remove_provisioning_network(task)
        task.driver.network.configure_tenant_networks(task)
```

(`ironic/drivers/modules/agent_base.py:1159-1170`). Read that literally: the
provisioning network is *removed* and the tenant network is *configured*, with
the node powered off in between if the driver requires it. That is the moment
the switch port stops being on the provisioning VLAN and starts being on the
tenant's. The mirror image happens at the start of a deploy —
`unconfigure_tenant_networks` then `add_provisioning_network`
(`ironic/drivers/modules/agent.py:513-514`).

### 5. DHCP is a *separate* setting from `network_interface`.

`[dhcp]/dhcp_provider` defaults to `neutron`
(`ironic/conf/dhcp.py:21-24`) and has three implementations, registered at
`pyproject.toml:43-46`:

- **`neutron`** (`ironic/dhcp/neutron.py:34`) — writes the PXE boot options onto
  the Neutron port as `extra_dhcp_opts` (`ironic/dhcp/neutron.py:90`). Neutron's
  own DHCP agent then serves them.
- **`dnsmasq`** (`ironic/dhcp/dnsmasq.py:27`) — writes plain files into
  `[dnsmasq]/dhcp_optsdir` and `dhcp_hostsdir`
  (`ironic/dhcp/dnsmasq.py:88-94`) and tags each node with a generated
  `dnsmasq_tag` (`dnsmasq.py:52-56`) so options apply per node. A local dnsmasq
  picks the files up. This is the standalone / Bifrost-style answer.
- **`none`** (`ironic/dhcp/none.py:19`) — every method is `pass`
  (`none.py:22-26`). Your infrastructure already serves DHCP correctly.

The pairing on the diagram is the *usual* combination, not a constraint: nothing
in the code stops you running `network_interface = flat` with
`dhcp_provider = dnsmasq`, and that is in fact a common standalone setup.

## Things worth remembering

- Three networks, three purposes: **BMC** (works with the node off),
  **provisioning** (PXE + IPA callbacks), **tenant** (the instance).
- `noop` and `none` are legitimate production choices, not placeholders.
- `flat` reuses one bind for provisioning and tenant — that is precisely why it
  cannot isolate tenants.
- `neutron`'s power comes from `local_link_connection`; if inspection never
  filled in `switch_id`/`port_id`, binding has nothing to program.
- The VLAN flip is a deploy step with priority 30, so it happens near the end of
  the deploy, after the disk is written.
