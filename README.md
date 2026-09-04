# VirtFusion SEV-SNP Hook

This repository contains a hook for VirtFusion that enables support for AMD SEV-SNP.

Used internally at Pipehost.net to enable SEV-SNP hypervisors.

## What it does

`server-before-disk-create-primary_php_0010_snp.event` is a VirtFusion hook. Before a VPS's primary disk is created, it:

1. Checks whether the local hypervisor is SEV-SNP capable by reading
   `/sys/module/kvm_amd/parameters/sev_snp`. If that isn't `Y`, the hook exits and does nothing.
2. Reads the server's configured memory from the event payload.
3. Calls the VirtFusion API to inject custom libvirt domain XML for the VM:
   - `memoryBacking` set to `memfd` with shared access (required for SEV-SNP).
   - `memtune` hard limit set to configured memory + 1024 MiB headroom.
   - `launchSecurity type='sev-snp'` block with `cbitpos`, `reducedPhysBits`, and `policy`.
   - An `os` loader override pointing at the AMDSEV OVMF firmware, which forces the VM into
     UEFI boot.
4. Logs the outcome to `/var/log/vf-snp-hook.log`.

## Caveats

- SEV-SNP is applied automatically by this hook based on whether the
  hypervisor itself is capable, not by any setting a user or admin can set in
  VirtFusion.

- The injected custom XML forces the VM to boot in UEFI, but VirtFusion's own
  UI/DB still believes the VPS boots in BIOS mode.

  Switching the Boot mode to UEFI in VirtFusion causes it to add its own additional
  UEFI-related XML lines to the libvirt config. Those lines conflict with the custom XML
  this hook already injected, and the VM will fail to boot.

- The host CPU/firmware must support SEV-SNP and the BIOS must be correctly configured for it

- The hook points the loader at `/usr/local/share/ovmf/ovmf.amdsev.fd` (see `OVMF_PATH` in the script). This file is not
  available by default. It must be obtained from the `ovmf-amdsev` package:
  <https://packages.debian.org/sid/ovmf-amdsev>. The `.deb` has to be downloaded and manually
  extracted, and the resulting `.fd` file placed at the exact path configured in the hook before
  SEV-SNP VMs can boot.

## Configuration

Before deploying, set the following constants at the top of the hook:

- `VF_URL`: the VirtFusion panel URL.
- `VF_TOKEN`: a VirtFusion API token with permission to set custom XML on servers.
- `OVMF_PATH`: path to the extracted `ovmf.amdsev.fd` (default:
  `/usr/local/share/ovmf/ovmf.amdsev.fd`).
- `CBITPOS`, `RPB`, `SNP_POLICY`: SEV-SNP launch security parameters for the host CPU.
- `HEADROOM_MIB`: extra memory (MiB) added to the `memtune` hard limit above the VPS's
  configured memory.
- `SNP_LOG`: path to the hook's log file.

This hook should be placed in `/home/vf-data/events/hypervisor/`

## Native support

There's an open feature request asking VirtFusion to add AMD SEV-SNP and Intel TDX support
natively: <https://virtfusion.featureos.app/p/add-amd-sev-snp-and-intel-tdx-options>. Native
support would obviously be a lot better than this hook. If you think this matters, go upvote it.
