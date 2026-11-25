NVIDIA Driver Upgrade W/O changing anything else! 
# ccbrc cluster: NVIDIA driver upgrade and CUDA 13 toolkit module

This procedure upgrades the NVIDIA driver on one compute node at a time, from
550.76 to 595.104.02, and then adds CUDA Toolkit 13.5.0 as a new environment
module beside the existing CUDA versions. The kernel, the OS, ZFS and all
existing software are left untouched.

Versions were checked against NVIDIA's site on 25 November 2026.

## Current Versions

| Item | Value |
| --- | --- |
| Front node | `ccbrc` (no NVIDIA driver; hosts `/software` on ZFS, shared to all nodes over NFS) |
| Compute nodes | `mannose`, `gag`, `glyco1`, `glyco2` |
| OS | Rocky Linux 9.3 (Blue Onyx) |
| Kernel | 5.14.0-362.24.1.el9_3.0.1.x86_64 |
| gcc / glibc | 11.4.1 / 2.34 |
| Scheduler | Slurm |
| Current driver | 550.76, installed with NVIDIA's `.run` installer, proprietary kernel modules, no DKMS |
| Current CUDA | 12.3 in `/software/apps/cuda/12.3` (target of `/usr/local/cuda`); 12.4 RPMs on `gag` |
| Existing modules | `cuda/12.3`, `cuda/12.4.1`, `amber/23`, `namd/3.0b7-GPU`, `namd/3.0a13-GPU`, `boltzgen/v0.3.1`, `boltzgen/v0.1.2`, `ligandMPNN/2025-05`, `mpich/4.0.3` |

## Target versions

| Component | Now | Target | Why |
| --- | --- | --- | --- |
| GPU driver (compute nodes) | 550.76 | 595.104.02 | NVIDIA's current Production Branch release. Supports all GPUs in the cluster. |
| CUDA toolkit (shared) | 12.3, 12.4 | 13.5.0 added | Latest CUDA 13 release. Installed beside the old versions, never replacing them. |
| Kernel, OS, ZFS | Rocky 9.3, 5.14.0-362.24.1 | unchanged | The driver needs kernel 4.15+ and glibc 2.27+, which are met. |


## What NOT to do

1. **Do not run `dnf update` and do not change the kernel.** ZFS on `ccbrc` is
   built by DKMS for kernel 5.14.0-362.24.1, and the NVIDIA module is built for
   the running kernel as well.
2. **Do not remove anything.** `/software/apps/cuda/12.3`, the CUDA 12.4 RPMs
   and the existing modules stay as they are.
3. **Keep `/usr/local/cuda` pointing at 12.3.** The new toolkit is opt-in
   through `module load`.
4. **Use the `.run` driver installer only.** That is how 550.76 was installed.
   Do not mix it with NVIDIA's RPM driver packages.

## Step 0: Pre-checks on the node

Run as root on the node to be upgraded.

```bash
hostname
nvidia-smi --query-gpu=name,driver_version --format=csv
uname -r
ls /usr/bin/nvidia-uninstall
rpm -qa | grep -i -E 'nvidia|cuda' | grep -v '^cuda-'
modinfo nvidia | grep -i '^license'
rpm -q kernel-devel-$(uname -r) gcc make elfutils-libelf-devel
mokutil --sb-state
systemctl get-default
```

| Check | Expected | If not |
| --- | --- | --- |
| `nvidia-uninstall` | File exists | The driver came from RPMs. Stop; this procedure does not apply to that node. |
| `rpm -qa` line | No output | Same as above: driver RPMs are present. |
| `modinfo` license | `NVIDIA` | `Dual MIT/GPL` means open modules; use `--kernel-module-type=open` in Step 3. |
| `rpm -q` line | Four installed packages | Install the missing one. `kernel-devel` must match `uname -r` exactly. |
| `mokutil` | `SecureBoot disabled`, or "EFI variables are not supported on this system" (legacy BIOS boot) | With Secure Boot enabled the module must be signed. Stop. |
| `systemctl get-default` | Either value | If `graphical.target`, Step 3 stops the desktop first. |

Results recorded for `mannose` on 25 November 2026: license `NVIDIA`, all four
packages installed, legacy BIOS boot (no Secure Boot), default target
`graphical.target`. The `nvidia-uninstall` check was not recorded and must be
confirmed in Step 3.

## Step 1: Download the installers

Run as root on `ccbrc`. This affects nothing that is running.

```bash
mkdir -p ~/installers
cd ~/installers

# New driver
wget https://download.nvidia.com/XFree86/Linux-x86_64/595.104.02/NVIDIA-Linux-x86_64-595.104.02.run

# Current driver, kept for rollback if we may need it later 
wget https://download.nvidia.com/XFree86/Linux-x86_64/550.76/NVIDIA-Linux-x86_64-550.76.run

# Get CUDA toolkit
For the CUDA toolkit, open <https://developer.nvidia.com/cuda-downloads>,
choose Linux, x86_64, Rocky, 9, **runfile (local)**, and copy the `wget`
command the page shows. The page builds the file name in the browser, so this
document writes it as `cuda_13.5.0_<...>_linux.run`.

## Step 2: Drain the node

Run as root on `ccbrc`. Draining stops Slurm from sending new jobs to the
node; jobs already running there continue.

```bash
NODE=mannose
scontrol update NodeName=$NODE State=DRAIN Reason="NVIDIA driver upgrade"
squeue -w $NODE
sinfo -N -n $NODE
```

Repeat the last two commands until `squeue` lists no jobs and `sinfo` shows
`drained`.

To end the remaining jobs instead of waiting:

```bash
scancel --nodelist=$NODE
```

This kills other users' jobs on that node, so warn them first.

## Step 3: Upgrade the driver

Run as root on the node, over `ssh`. Start only when Step 2 shows the node
drained.

**3.1 Confirm the node is idle and nothing is running.**


**3.2 Stop the desktop and the driver's helper service.**

```bash
systemctl isolate multi-user.target
systemctl stop nvidia-persistenced 2>/dev/null
```

The first command is needed when the node boots to `graphical.target`, because
the desktop holds the driver open. It ends any desktop session on the node's
own screen; `ssh` sessions stay connected.

**3.3 Run the installer.** It finds 550.76 and removes it before installing.

```bash
sh /software/installers/NVIDIA-Linux-x86_64-595.104.02.run --kernel-module-type=proprietary
```

The flag matters: this installer defaults to the open kernel modules, and the
cluster runs the proprietary ones. If the shell reports "permission denied",
copy the file to `/root` and run it from there.

**3.4 Answer the prompts.**

| Prompt | Answer |
| --- | --- |
| Existing driver found, continue? | Yes |
| Install 32-bit compatibility libraries? | No |
| Register with DKMS? | No |
| Rebuild the initramfs? | Yes |
| Update the X configuration file? | No |

Not every prompt appears on every system. If the build fails, the installer
writes `/var/log/nvidia-installer.log`. Stop there and go to


**3.5 Reboot.** The node returns to its default target by itself.

```bash
reboot now
```

## Praise for system to come back on network .. I am more afraid of breaking anything else:)

## Step 4: Verify and test the old software

Run on the node after it comes back.

```bash
nvidia-smi
cat /proc/driver/nvidia/version
systemctl status slurmd --no-pager
module load cuda/12.3
nvcc --version
module purge
```

`nvidia-smi` should show `Driver Version: 595.104.02` and list the node's GPU,
and `slurmd` should be active.

The node is still drained, so start the test jobs directly over `ssh`. Use
short jobs that have been run before, so a correct result is recognizable.


## Step 5: Return the node to Slurm

Run as root on `ccbrc` once every test in Step 4 passes.

```bash
NODE=mannose
scontrol update NodeName=$NODE State=RESUME
sinfo -N -n $NODE
srun -w $NODE --gres=gpu:1 nvidia-smi
```

`sinfo` should show `idle`, and the last command should print the new driver
version from inside a Slurm job. Replace `--gres=gpu:1` with the GPU request
the cluster's job scripts normally use.


## Just stop here! I can test installing RD4, and if that works, we don't need to do the following steps, and update the toolkit when needed. 

## Step 6: Add CUDA 13.5.0 as a module (I can do that by myself later on!)

Run once, as root on `ccbrc`. It only copies files into
`/software/apps/cuda/13.5.0`. No reboot is needed and no node's driver is
touched.

**6.1 Record the current defaults.**

```bash
readlink /usr/local/cuda
ls /etc/ld.so.conf.d/ | grep -i cuda
```

**6.2 Install the toolkit (interactive).**

```bash
cd /software/installers
sh cuda_13.5.0_<...>_linux.run
```

In the installer menu:

1. Accept the license.
2. Untick **Driver** if it is listed. From CUDA 13.4 on NVIDIA no longer
   bundles the driver, so it may not appear.
3. Keep **CUDA Toolkit** ticked. Untick extras that are not needed (demo
   suite, documentation, kernel objects).
4. Open the toolkit's advanced options (highlight CUDA Toolkit, press `A`):
   - change the install path to `/software/apps/cuda/13.5.0`
   - untick "Create symbolic link from /usr/local/cuda"
   - untick desktop shortcuts and man pages
5. Choose **Install**.

The menu layout is described from the CUDA 12.x installers; the 13.5.0 screens
may differ slightly.

Silent alternative: run `sh cuda_13.5.0_<...>_linux.run --help` first to
confirm the option names, then for example:

```bash
sh cuda_13.5.0_<...>_linux.run --silent --toolkit --toolkitpath=/software/apps/cuda/13.5.0 --no-man-page
```

**6.3 Check the result and the defaults.**

```bash
ls /software/apps/cuda/13.5.0/bin/nvcc
readlink /usr/local/cuda
ls /etc/ld.so.conf.d/ | grep -i cuda
```

`readlink` must still print `/software/apps/cuda/12.3/`. If it changed:

```bash
ln -sfn /software/apps/cuda/12.3/ /usr/local/cuda
```

If a new `cuda-13*` file appeared in `/etc/ld.so.conf.d/`, delete it and run
`ldconfig`.

**6.4 Add the module file.** The first line of `module show` prints the path
of the existing module file.

```bash
module show cuda/12.3
cp <module file path shown above> <same folder>/13.5.0
sed -i 's/12\.3/13.5.0/g' <same folder>/13.5.0
module show cuda/13.5.0
```

If the module file ends in `.lua`, name the copy `13.5.0.lua`. All paths in
the last output should be under `/software/apps/cuda/13.5.0`.

## Step 7: Test the new toolkit

Run on a node that already has the new driver. This compiles and runs a small
program, which tests the compiler and the driver pairing together.

```bash
module load cuda/13.5.0
nvcc --version
cd /tmp
cat > hello.cu <<'EOF'
#include <stdio.h>
#include <cuda_runtime.h>
int main(){int n=0; cudaError_t e=cudaGetDeviceCount(&n); printf("err=%d devices=%d\n",(int)e,n); return (int)e;}
EOF
nvcc hello.cu -o hello && ./hello
```

| Result | Meaning |
| --- | --- |
| `err=0 devices=1` or more | CUDA 13.5.0 works with gcc 11.4.1 and driver 595.104.02. |
| Compile error | The toolkit does not accept gcc 11.4.1. |
| Non-zero `err` | Driver 595 is too old for this toolkit. Install CUDA 13.2.2 as the module instead, or move to the 620 driver branch. |

The `cuda/13.5.0` module appears on every node because `/software` is shared,
but programs built with it run only on nodes that already have the new driver.

## Rollback

A node returns to its starting state by reinstalling the old driver. Run as
root on the node.

```bash
systemctl isolate multi-user.target
systemctl stop nvidia-persistenced 2>/dev/null
sh /software/installers/NVIDIA-Linux-x86_64-550.76.run
reboot
```
