# 07 — Images, config-drive and inspection data, in plain English

Diagram: `07-data-and-artifacts.html` (spec `07-data-and-artifacts.dataflow.json`)

All citations are `path:line` inside the pinned clone
`.src-pinned/ironic` at revision `26c022f1353e40f36456905620e8c1aca8cf1a31`.

## The one-paragraph mental model

Everything Ironic puts on a machine is a **file it had to go and fetch**, and
everything it learns about a machine is a **payload the machine sent back**. On
the way out: a kernel and ramdisk to network-boot with, an operating-system image
to write to disk, and optionally a config-drive so cloud-init has something to
read. On the way back: an inspection inventory (which becomes node properties and
port rows), and, when something goes wrong, the ramdisk's own logs. The
conductor sits in the middle. It is the only process that downloads, verifies,
caches and serves those files — the node never talks to Glance.

## The five things that confuse first-time readers

### 1. `image_source` is one field with four completely different back ends.

There is no "image driver" setting. Ironic looks at the **URL scheme** and picks a
class from a plain dictionary (`ironic/common/image_service.py:1298`):

```python
protocol_mapping = {
    'http': HttpImageService,   'https': HttpImageService,
    'file': FileImageService,   'glance': GlanceImageService,
    'oci': OciImageService,     'nfs': NfsImageService,
    'cifs': CifsImageService,   'smb': CifsImageService,
}
```

`get_image_service()` (`ironic/common/image_service.py:1310`) parses the scheme
and looks it up. The one special case: **a bare, scheme-less value is only
accepted if it looks like a UUID, and then it means Glance**
(`ironic/common/image_service.py:1324-1332`). Anything else scheme-less is
rejected outright. This is why "I pasted an image name and got
`ImageRefValidationFailed`" is such a common first bug.

Glance is not magic either — Ironic asks it for a temporary URL and then behaves
like a plain HTTP client (`get_temp_url_for_glance_image`,
`ironic/common/images.py:628`).

### 2. The checksum is verified by the conductor, not by the node.

`images.fetch()` (`ironic/common/images.py:447`) downloads and then, unless the
transfer already proved a checksum for itself, validates the file on disk:

```python
if (not transfer_checksum
        and not CONF.conductor.disable_file_checksum
        and checksum):
    checksum_utils.validate_checksum(path, checksum, checksum_algo)
```

(`ironic/common/images.py:455-458`, implementation at
`ironic/common/checksum_utils.py:54`). The `checksum` field may itself be a URL
to a `SHA256SUMS`-style file — `is_checksum_url`
(`ironic/common/checksum_utils.py:196`) and `get_checksum_from_url`
(`ironic/common/checksum_utils.py:212`) handle that.

Some services prove the checksum during transfer instead (OCI registries pin a
digest), which is what `transfer_verified_checksum` at
`ironic/common/images.py:407-415` means.

### 3. The image cache is a hard-link farm, and that is what makes eviction work.

`ImageCache` (`ironic/drivers/modules/image_cache.py:51`) downloads once into a
**master path** and then hard-links from there to wherever the file is needed:

```python
os.link(tmp_path, master_path)
os.link(master_path, dest_path)
```

(`ironic/drivers/modules/image_cache.py:251-252`; the reuse path is
`image_cache.py:171`). Two master directories exist by default:
`/var/lib/ironic/master_images` for instance images
(`ironic/conf/pxe.py:41`) and `/tftpboot/master_images` for TFTP artifacts
(`ironic/conf/pxe.py:91`). Setting either to the empty string disables caching.

Eviction is elegant: a cached file is only deletable if **nothing links to it**.

```python
if not os.path.isfile(filename) or stat.st_nlink > 1:
    continue
```

(`ironic/drivers/modules/image_cache.py:393`, in
`_find_candidates_for_deletion`). So an image currently in use by a deploy can
never be evicted, without any bookkeeping. Size and age caps are
`[pxe]/image_cache_size` (default 20480 MiB, `ironic/conf/pxe.py:46`) and
`image_cache_ttl`.

Kernel and ramdisk go through the same machinery: `cache_ramdisk_kernel`
(`ironic/common/pxe_utils.py:1312`) creates a per-node directory under
`[deploy]/http_root` for iPXE or `[pxe]/tftp_root` for classic PXE
(`pxe_utils.py:1318-1320`) and calls `fetch_images(..., TFTPImageCache(), ...)`
(`pxe_utils.py:1346`).

### 4. The config-drive is sometimes an ISO Ironic builds, and sometimes just a URL.

`get_configdrive_image` (`ironic/conductor/configdrive_utils.py:531`) is the whole
decision:

```python
configdrive = node.instance_info.get('configdrive')
if isinstance(configdrive, dict):
    configdrive = build_configdrive(node, configdrive)
return configdrive
```

- If you passed a **dict** of `meta_data` / `network_data` / `user_data` /
  `vendor_data`, Ironic builds a gzipped, base64-encoded ISO9660 image
  (`build_configdrive`, `ironic/conductor/configdrive_utils.py:503`), defaulting
  `meta_data['uuid']` to the node UUID.
- If you passed a **string**, it is returned untouched — that can be an
  already-built base64 blob **or a plain URL the agent will fetch itself.**

Either way the result is handed to the agent inside `image_info`
(`ironic/drivers/modules/agent.py:725-729` for the ramdisk deploy path,
`agent.py:966-987` for the standard one). Ironic can also crack open a
user-supplied ISO and re-master it when the embedded network metadata is stale
(`check_and_patch_configdrive`, `ironic/conductor/configdrive_utils.py:68`).

### 5. Inspection data does not stay in one place.

When the ramdisk posts its inventory, `continue_inspection`
(`ironic/conductor/inspection.py:119`) runs the inspection **hooks**, applies
inspection rules, drops the (huge) logs from the plugin data, and only then
stores the result (`ironic/conductor/inspection.py:150-156`). Storage is
configurable — `store_inspection_data`
(`ironic/drivers/modules/inspect_utils.py:130`) writes to `none`, to the
**database** as a `NodeInventory` row (`inspect_utils.py:147-152`), or to
**Swift** (`inspect_utils.py:155-159`).

Meanwhile the hooks have already written things to normal Ironic tables:

- `PortsHook` creates a `ports` row per validated NIC —
  `objects.Port(task.context, **port_dict)` then `port.create()`
  (`ironic/drivers/modules/inspector/hooks/ports.py:37-48`), carrying the MAC and
  the `pxe_enabled` flag.
- `ParseLLDPHook` decodes the raw LLDP TLVs into `plugin_data['parsed_lldp']`
  (`ironic/drivers/modules/inspector/hooks/parse_lldp.py:27-32`).
- `LocalLinkConnectionHook` turns LLDP into the switch identity Ironic needs for
  Neutron port binding: `port.set_local_link_connection(...)` with `switch_id`
  and `port_id` (`.../hooks/local_link_connection.py:27-28`, `:92-94`).

So "inspection results" is really three destinations: node properties, the
`ports` table, and a separate inventory blob.

### Bonus: ramdisk logs are pulled, not pushed.

`collect_ramdisk_logs` (`ironic/drivers/utils.py:350`) issues a
`collect_system_logs` command *to the agent* and stores what comes back. It is a
no-op when `[agent]/deploy_logs_collect = never` (`ironic/conf/agent.py:63`), and
the destination is either a local directory or a Swift container
(`ironic/drivers/utils.py:323` and `:334`, configured at
`ironic/conf/agent.py:72-98`). It fires on cleaning failure at
`ironic/conductor/cleaning.py:334`.

## Things worth remembering

- The node never authenticates to Glance. The conductor fetches; the node gets a
  URL on the provisioning network (or a file over TFTP/HTTP).
- If a deploy fails "checksum mismatch", it failed on the **conductor's** disk.
- Hard-link counts are the cache's whole reference-counting scheme.
- A config-drive that is a URL is never inspected by Ironic, so a broken URL
  fails inside the ramdisk, not in the API.
- Inspection writes to `ports` immediately; do not expect to find the MACs only
  in the inventory blob.
