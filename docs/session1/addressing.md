# Session 1 — Platform Setup

## GNS3 Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| GNS3 | 2.2.44 | Latest stable |
| GNS3 VM | 2.2.44 (must match GNS3) | Latest stable |
| Hypervisor | VMware Workstation Player (free) | VMware Workstation Pro |
| GNS3 VM RAM | 8 GB | 16 GB |
| GNS3 VM vCPUs | 4 | 8 |
| GNS3 VM Disk | 50 GB | 100 GB |

## vMX Image

| Property | Value |
|----------|-------|
| Image name | `hda.qcow2` (vMX 14.1R4.8) |
| Source | [support.juniper.net](https://support.juniper.net) — free with Juniper account |
| Format | QCOW2 (QEMU disk image) |
| Disk size | ~2 GB |
| RAM per node | 2048 MB |
| vCPUs per node | 1 |
| Boot time | 3–5 minutes |

## QEMU Template Settings

| Setting | Value |
|---------|-------|
| RAM | 2048 MB |
| vCPUs | 1 |
| HDD interface | `ide` |
| NIC type | `virtio-net-pci` |
| NIC count | 6 |
| Additional QEMU options | `-serial mon:stdio -nographic -M pc` |
| KVM | Required |

## Adapter to Interface Mapping

| GNS3 Adapter | Junos Interface | Notes |
|---|---|---|
| Adapter 0 | `em0` | External management |
| Adapter 1 | `em1` | Internal (pre-configured 172.16.0.1/16) |
| Adapter 2 | `ge-0/0/0` | First lab interface |
| Adapter 3 | `ge-0/0/1` | Second lab interface |
| Adapter 4 | `ge-0/0/2` | Third lab interface |
| Adapter 5 | `ge-0/0/3` | Fourth lab interface |

!!! warning "Always connect Adapter 2+ for lab links"
    Adapters 0 and 1 are reserved by vMX for management. All GNS3 topology links must use Adapter 2 or higher.
