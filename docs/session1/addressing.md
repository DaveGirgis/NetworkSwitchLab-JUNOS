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

## vJunos-router Image

| Property | Value |
|----------|-------|
| Image name | `vjunos-router-23.2R1.15.qcow2` (or later) |
| Source | [support.juniper.net](https://support.juniper.net) — free with Juniper account |
| Format | QCOW2 (QEMU disk image) |
| Disk size | ~5 GB |
| RAM per node | 4096 MB |
| vCPUs per node | 2 |
| Boot time | 3–5 minutes |

## Download Steps

1. Create a free account at [support.juniper.net](https://support.juniper.net)
2. Navigate to **Downloads** > **Software** > **Junos Platforms** > **Virtual** > **vJunos-router**
3. Select the latest **23.2R1** (or higher) release
4. Download the `.qcow2` file
5. Place the file in a memorable location — you will point GNS3 to it during import

!!! note "No license required"
    vJunos-router does not require a license file for lab use. All JNCIS-SP features (OSPF, IS-IS, BGP, MPLS, VPN) are available in the evaluation image.
