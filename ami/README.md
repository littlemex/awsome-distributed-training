<!-- markdownlint-disable MD013 -->

# Amazon Machine Image catalog

This directory builds 6 x86_64 AMIs with Packer and Ansible. Each `ami_*` target listed below builds 1 image. AMI builds launch billable EC2 resources, so the aggregate `ami` target exits without building anything.

## Prerequisites

Install GNU Make, [Packer](https://developer.hashicorp.com/packer/install), Ansible, and AWS credentials that can read public SSM parameters, call EC2 `DescribeImages`, and create AMIs. Initialize the plugins before the first build:

```bash
packer init packer-ami.pkr.hcl
```

Select one target explicitly. `AWS_REGION` defaults to `us-east-1`.

```bash
AWS_REGION=us-east-1 make ami_eks_ubuntu2404
```

Public SSM parameters provide 5 parent AMIs. The ParallelCluster parent is selected with EC2 `DescribeImages`. A later build from the same Git revision can therefore use a newer parent. The resulting AMI tags record the resolved parent AMI ID and lookup family; retain the Packer log as build evidence.

## Active AMIs

| Make target | Parent | Custom layer and ownership | Base AMI release notes |
| --- | --- | --- | --- |
| `ami_ec2_ubuntu2404` | Canonical Ubuntu Server 24.04 LTS | EFA 1.50.0. The workload owns CUDA, NCCL, and frameworks. | [Canonical EC2 image discovery](https://documentation.ubuntu.com/aws/aws-how-to/instances/find-ubuntu-images/) |
| `ami_ec2_ubuntu2404_lustre` | Canonical Ubuntu Server 24.04 LTS | Pins the `linux-aws-lts-24.04` kernel line, reboots into it, then installs EFA 1.50.0 in minimal mode and an FSx for Lustre client 2.15.6 built from source for that kernel release. Mounting a file system is left to runtime configuration. | [Canonical EC2 image discovery](https://documentation.ubuntu.com/aws/aws-how-to/instances/find-ubuntu-images/) and [Lustre client compatibility](https://docs.aws.amazon.com/fsx/latest/LustreGuide/lustre-client-matrix.html) |
| `ami_ec2_ubuntu2404_dlami` | Base OSS NVIDIA Driver GPU DLAMI, Ubuntu 24.04 | Validates the parent NVIDIA driver and does not replace parent libraries. | [AWS Deep Learning Base GPU AMI release notes](https://docs.aws.amazon.com/dlami/latest/devguide/appendix-ami-release-notes.html) |
| `ami_pcluster_ubuntu2404` | AWS ParallelCluster 3.15.1, Ubuntu 24.04 | Retains the official ParallelCluster software stack and validates its image marker. | [AWS ParallelCluster 3.15.1 release notes](https://github.com/aws/aws-parallelcluster/wiki/3.15.1) |
| `ami_eks_al2023` | EKS 1.35 optimized AL2023 standard | Installs NVIDIA driver 580.126.09 from the AWS-maintained AL2023 NVIDIA repository. Kubernetes bootstrap remains owned by `nodeadm`. | [Amazon EKS AMI releases](https://github.com/awslabs/amazon-eks-ami/releases) and [AL2023 NVIDIA advisory](https://alas.aws.amazon.com/AL2023/ALAS2023NVIDIA-2026-271.html) |
| `ami_eks_ubuntu2404` | Canonical Ubuntu 24.04 EKS 1.35 | Installs EFA 1.50.0 in minimal mode: EFA kernel module and `rdma-core` only. GPU Operator owns NVIDIA; workload images own CUDA, NCCL, Libfabric, MPI, and aws-ofi-nccl. | [Canonical EKS image discovery](https://documentation.ubuntu.com/aws/aws-how-to/instances/find-ubuntu-images/) and [EFA release notes](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-changelog.html) |
| `ami_eks_ubuntu2404_lustre` | Canonical Ubuntu 24.04 EKS 1.35 | Same Lustre and EFA layer as `ami_ec2_ubuntu2404_lustre`. Kubernetes bootstrap remains owned by the parent image. | [Canonical EKS image discovery](https://documentation.ubuntu.com/aws/aws-how-to/instances/find-ubuntu-images/) and [Lustre client compatibility](https://docs.aws.amazon.com/fsx/latest/LustreGuide/lustre-client-matrix.html) |
| `ami_pcs_ubuntu2404` | Base OSS NVIDIA Driver GPU DLAMI, Ubuntu 24.04 | Adds AWS PCS Agent 1.5.1-1 and AWS PCS Slurm 25.11.7-3 with official installers. Use this AMI with the Slurm 25.11 series. `architectures/aws-pcs` installs Enroot/Pyxis at first boot when `InstallEnrootPyxis=true`. | [DLAMI release notes](https://docs.aws.amazon.com/dlami/latest/devguide/appendix-ami-release-notes.html), [AWS PCS custom AMI guide](https://docs.aws.amazon.com/pcs/latest/userguide/working-with_ami_custom.html), and [installer checksums](https://docs.aws.amazon.com/pcs/latest/userguide/working-with_ami_installers.html#working-with_ami_installers_verify) |

The Canonical links describe image discovery rather than per-build release notes. Record the resolved AMI ID and Canonical serial from EC2 image metadata for each build.

## Component ownership

Parent-provided components are not reinstalled. The ParallelCluster and Base GPU DLAMI parents retain their NVIDIA, EFA, CUDA, and communication-library stacks. The PCS custom layer adds only the PCS Agent and Slurm. It intentionally excludes Enroot and Pyxis. Keep `architectures/aws-pcs` parameter `InstallEnrootPyxis=true` unless another layer provides both components.

EFA Installer 1.50.0 full mode includes aws-ofi-nccl 1.21.1. The minimal EKS Ubuntu build excludes aws-ofi-nccl. This catalog does not build or install a separate host NCCL stack; workload images own NCCL, including the NCCL 2.31.2-1 compatibility target. NCCL benchmarks remain under `micro-benchmarks/nccl-tests`.

## FSx for Lustre client

The client is a kernel module, and the published modules are named per exact kernel release.
Two consequences shape the Lustre targets.

The Ubuntu 24.04 server and EKS parents boot a rolling `linux-aws` kernel whose series has no
published client module, so the build pins the `linux-aws-lts-24.04` kernel line and reboots
into it before the client is installed. That is why the Lustre builds run two Ansible stages
with a reboot between them.

For the pinned kernel release the repository publishes no binary module either, so the module is
built from the `lustre-source` package during the image build and installed as a package. The
image then carries a module that matches its kernel, and a build that fails fails the image build
rather than a node. `aws_lustre_mode: dkms` registers the source instead, so the module follows
later kernel installs on a host that updates in place; that is the right mode for long-lived nodes
and the wrong one for an immutable image, because it moves compilation onto the fleet.
`aws_lustre_mode: binary` installs a published module, which exists for some kernel releases and
not for the pinned one, so that mode currently refuses on this line and says why.

`roles/aws_lustre/files/lustre_installer.sh` carries the Ubuntu procedure, and the role stages it
and calls it, in the same shape as the EFA role calling `efa_installer.sh`. The script also runs
standalone on a host this repository did not build, so the packaging, build and verification
steps are one implementation rather than two. The role adds what an image build needs around it:
pinning a kernel line, the reboot, and its own assertions afterwards.

```bash
./lustre_installer.sh -y                     # DKMS, verify, print a summary
./lustre_installer.sh -y --mode binary       # published module for the running kernel
./lustre_installer.sh --check                # does the running kernel have a usable client
./lustre_installer.sh --uninstall -y
```

Two details cost time to find. The `lustre-source` package does not declare `flex`, `bison` or
the Python headers, which `configure` needs, so the installer adds them. And the userspace helper
from `lustre-client-utils` is required: without `/sbin/mount.lustre` the kernel receives the raw
option list and refuses the mount, which reads like a module problem and is not one.

The client source does not build against every kernel: 2.15.6 fails on the 7.x series. That
matters for the dkms mode, and the failure mode was measured rather than assumed. Installing a 7.x
kernel on a host with an unfiltered DKMS registration ends with `apt` returning 100 and the kernel
package left `half-configured`, which blocks later package operations until `dpkg --configure`
runs. With `aws_lustre_kernel_filter` in place the same install returns 0, DKMS skips the kernel,
and no module is produced for it. Skipping keeps package management healthy but leaves a node that
boots without a client, which a mount would be the first thing to notice, so the installer also
installs `lustre-client-check.service`. It runs `--check` at boot and reports the state; it does
not block boot, because a node that refuses to start is worse than one that reports a fault.
Wiring that result into scheduling, for example a taint applied by a node agent, and gating a
kernel rollout on a successful build, are the operator's responsibility and are not provided
here.

The role verifies what it produced. It asserts that the module `vermagic` equals the running
kernel release, loads the module, and requires `libcfs`, `lnet` and `lustre` to be resident. It
does not mount a file system: a mount target belongs to runtime configuration, not to an image.

On Amazon Linux 2023 none of this applies, because the kernel package itself provides
`kmod-lustre-client`. The role installs only `lustre-client` there and reports the provider.
Adding `aws_lustre` to `playbook-eks-al2023.yml` is therefore a one-line change with no kernel
pinning and no reboot.

## Target migration

Old target names are not aliases.

| Previous target | Replacement |
| --- | --- |
| `ami_base` | `ami_ec2_ubuntu2404` |
| `ami_dlami_gpu` | `ami_ec2_ubuntu2404_dlami` |
| `ami_pcluster_gpu` | `ami_pcluster_ubuntu2404` |
| `ami_eks_gpu` | `ami_eks_al2023` |

The three retired Neuron and CPU variants have no replacement.

## Validation boundary

Packer formatting, configuration validation, Ansible syntax checks, lint checks, and dry-run Make targets are desk checks. They do not prove that an AMI builds or works on target hardware. Before publishing an AMI, build the selected target in an approved AWS account and validate its runtime contract on the intended EC2, EKS, ParallelCluster, or PCS deployment.
