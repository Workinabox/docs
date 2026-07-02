# Sandbox VMs (Firecracker)

Architecture of record for the microVM sandboxes agents run in, and the backend `vm` module
that creates and runs them. Companion to [DDD.md](DDD.md) and [AGENT_MODEL.md](AGENT_MODEL.md).

This iteration delivers **headless, role-templated Firecracker microVMs** plus the backend
module to create/run/stop/list them. GUI, the in-guest agent runtime, agent comms, and teams
are deferred — see [Future direction](#future-direction).

## Why Firecracker

`demo.workinabox.ai` is a single Ubuntu 24.04 host VM (provisioned by the `iac/` repo on
xcp-ng with **nested virtualization**) that already has **Firecracker + jailer** installed and
KVM smoke-tested at first boot. Agents need disposable, isolated Linux environments to work in;
Firecracker gives each one a hardware-isolated microVM that boots in well under a second from a
shared read-only image.

### Two nested VM levels — keep them straight

- **Demo VM** — the xcp-ng host; runs the `wiab` backend; deploy-time lifecycle via terraform.
- **Firecracker microVMs** — the sandboxes booted *on* the demo VM at runtime, many per day.
  From the host each is **one jailed process**; inside each is a full headless Linux
  (systemd → sshd + tools).

## Firecracker mechanics (how a microVM is made)

A running microVM is one userspace `firecracker` process using `/dev/kvm`. There is no big
"create" call — you spawn the process and configure it over a tiny REST API on a Unix socket:

1. **Ingredients on disk:** the `firecracker` binary, a guest kernel (`vmlinux`, uncompressed),
   and a rootfs as a **raw block device** (`developer.ext4` — a flat file whose bytes are
   exactly a disk partition; Firecracker does *not* understand qcow2/VMDK).
2. **Host setup first:** create a **tap device** (a virtual NIC — one end is `tap0` on the
   host, the other is `eth0` in the guest) and a per-instance writable overlay file.
3. **jailer:** production runs `jailer`, which builds a chroot + cgroups + namespaces, drops
   privileges, then execs `firecracker` inside that cage. The rootfs, kernel, and API socket
   must live inside the jail's chroot dir.
4. **Configure over the socket** (`--api-sock`): HTTP `PUT`s carrying JSON —
   `PUT /boot-source` (kernel + `boot_args` incl. `ip=` for guest networking),
   `PUT /drives/rootfs` (read-only), `PUT /drives/scratch` (writable overlay),
   `PUT /machine-config` (vcpus, mem), `PUT /network-interfaces/eth0` (host tap + guest MAC).
5. **Boot:** `PUT /actions {"action_type":"InstanceStart"}`.
6. **Stop:** `SendCtrlAltDel` (graceful) or kill the PID, then tear down the tap + overlay.

## Image model — layered base + role templates

Root filesystems are built as **layered OCI images exported to ext4** (`iac/images/`):

- `base/Dockerfile` — `FROM ubuntu:24.04`, headless: `systemd` (init), `openssh-server`, and
  common CLI tooling (`git`, `curl`, `python3`, …). No desktop/X/VNC.
- `developer/Dockerfile` — `FROM wiab-vm-base`, adds Rust + `build-essential`. **Roles are just
  child images** — this is the "base extended per role" mechanism.
- `build.sh` — `docker build` → `docker export` → pack into a fixed-size raw **ext4** via
  `mke2fs -d`. Produces `base.ext4`, `developer.ext4` (read-only at runtime).
- `kernel/fetch.sh` — pins a Firecracker-compatible `vmlinux`.

### Distribution — CI → Azure blob → azcopy

Images are large, change rarely, and can't live in git. They are built in **CI**, uploaded to
**Azure blob**, and pulled onto the demo VM with `azcopy` into `/var/lib/wiab/images/` — the
same mechanism the LLM model weights already use.

> **Open risk:** the pinned kernel must include `overlayfs` (+ `virtio-net`, `virtio-blk`,
> `ext4`). Verify before relying on the overlay run model; a custom kernel build may be needed.

## Per-instance run model

Booting a sandbox must **not** copy a multi-GB image:

- `/dev/vda` = the role image (`developer.ext4`), **read-only**, shared by all instances.
- `/dev/vdb` = a small **writable overlay** ext4, created per instance.
- Guest init uses **overlayfs** (lower = read-only image, upper = overlay) so only per-instance
  deltas are written.
- Networking: the host allocates a **tap + IP** per VM; static IP is passed via the kernel
  `ip=` boot arg; the VM is reachable (e.g. `sshd`) on that IP.

## Backend `vm` module

A DDD bounded context mirroring the `agent` module (core → app → inf). **Control surface is
internal-only — there is no external HTTP route.** VMs are launched by backend code; the
persisted aggregate lets running sandboxes be listed and reconciled after a restart.

- **`wiab-core::vm`** — `Vm` aggregate (state machine `Creating → Running → Stopped`/`Failed`),
  value objects `VmId` (`VM-###`), `VmState`, `VmTemplate`, `VmResources`, the `VmRepository`
  and `VmNumbering` ports, `VmError`, `VmSnapshot`.
- **`wiab-app`** — `VmApplicationService` (`provision_vm`, `stop_vm`, `get_vm`, `list_vms`) and
  the **`VmRuntime`** port (`launch(VmSpec) -> RuntimeHandle`, `shutdown`). `provision_vm`
  persists `Creating`, asks the runtime to boot, then records `Running` (+ guest IP) or `Failed`.
- **`wiab-inf`** — `PostgresVmRepository` + `InMemoryVmRepository` + `VmRepo` enum dispatch
  (migration `V12__vm.sql`); `InMemoryVmNumbering`; **`InMemoryVmRuntime`**.

The **hypervisor is a seam** — the `VmRuntime` trait. `InMemoryVmRuntime` is its in-memory
implementation (the analogue of the `InMemory*` repositories): it boots nothing real but
models VM lifecycle in memory, so the module runs and is testable on hosts without KVM. The
real **`FirecrackerRuntime`** (tap + overlay + jailer + the socket `PUT` sequence above) is a
second implementation behind the same trait, added and boot-verified on the demo VM.

The service is exercised today via tests (provision → get → stop against the in-memory repo +
runtime). Its production caller is the agent/team orchestration — future work.

## Future direction

Recorded so the reasoning isn't lost; **not built** in this iteration.

- **GUI / remote desktop.** A software desktop inside the guest — Xvfb + a window manager +
  XFCE + x11vnc + websockify/noVNC — reached over virtio-net so agents *and* humans can
  view/control the screen (computer-use). Firecracker has no graphics device, so this is the
  only path; it is software-rendered (not GPU accelerated). Layers onto the headless base as
  extra packages + guest services.
- **In-guest agent runtime.** An "agent launcher" *slot* in the base (systemd service that
  brings up net, reads a model-endpoint URL, execs an agent binary), with the fast-changing
  **agent binary decoupled** from the GB image and delivered via a read-only `/dev/vdc` "agent
  drive" (host holds `current.ext4`, updated at deploy time; every microVM picks up the latest
  at boot — no image rebuild). Model calls go *out* of the guest; tool calls stay *in* it.
- **Comms — hub-and-spoke over vsock.** Agents never talk peer-to-peer. Each microVM gets a
  privileged **virtio-vsock** channel to a backend broker (identity by construction — no
  tokens); the backend brokers *both* agent↔backend and agent↔agent. General internet egress is
  a separate NAT data plane. (Maps to [AGENT_MODEL.md](AGENT_MODEL.md) delegate/consult/escalate.)
- **Teams.** The real unit of execution is a **team** — an orchestrator agent + member agents,
  each in its own role-templated microVM, linked by team-scoped comms. Needs a `Team` aggregate +
  a `provision_team` flow. Distinguish *work* orchestration (the orchestrator agent) from
  *infrastructure* orchestration (the backend). `Vm` would gain `team_id` + `role`.
