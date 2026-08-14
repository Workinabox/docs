# Installing sandbox VM images

Agent sandboxes boot from prebuilt root filesystem images. They are large, change rarely, and
cannot live in git — so they are built in CI, uploaded to Azure blob storage, and pulled onto the
host with `azcopy`. This is the same mechanism the LLM model weights use.

This page covers **getting the images onto a host**. For how the images are built and what is in
each layer, see [`iac/images/README.md`](../../iac/images/README.md), which is maintained
alongside the build scripts.

## What gets distributed

The default artifact set (`vm_image_files`):

| File | What it is |
| --- | --- |
| `base.ext4` | Base rootfs — systemd init, sshd, common CLI tooling. |
| `developer.ext4` | Extends base with the Rust toolchain and `build-essential`. |
| `vmlinux` | The pinned guest kernel. |
| `initramfs` | Assembles the overlay root and switches into systemd. |

Role images are just child images of the base — adding a role means adding a template, not a new
distribution mechanism.

At runtime the role image is mounted **read-only** and shared by every microVM on the host; each
instance gets a small writable overlay, so booting a sandbox never copies a multi-GB file.

> The guest kernel must include `overlayfs`, `virtio-net`, `virtio-blk` and `ext4`. This is why
> the kernel is built and pinned rather than taken from the distro — `images/kernel/build.sh`
> produces it. (`images/kernel/fetch.sh`, which pulled a prebuilt Firecracker CI kernel, is
> legacy.)

## Building and uploading (CI)

The **Build sandbox VM images** workflow (`iac/.github/workflows/images.yml`) does this. It runs
automatically when anything under `images/` changes on `main`, and can be dispatched manually to
choose which templates to build and whether to rebuild the kernel.

Two repository secrets are required:

| Secret | Purpose |
| --- | --- |
| `WIAB_IMAGES_URL` | Azure blob container URL **with an embedded SAS token**. The upload target. |
| `WIAB_GUEST_SSH_PUBKEY` | Demo login key baked into the guest `wiab` user. May be empty — the guest then has no `authorized_keys`. |

Building locally is possible (`images/build.sh <template>`) but the kernel and initramfs steps
need Linux. The build scripts only produce artifacts; they never upload.

## Fetching onto a host

`wiab-deploy` fetches images on first boot and on every in-place deploy. Configure it through
Terraform:

| Variable | Default | Notes |
| --- | --- | --- |
| `wiab_images_url` | `""` | Azure blob container URL with an embedded SAS token. **Empty disables image fetching entirely.** |
| `vm_images_version` | — | Change this to trigger a re-fetch. |
| `vm_image_files` | `["base.ext4", "developer.ext4", "vmlinux", "initramfs"]` | Filenames to fetch. |
| `wiab_data_dir` | `/var/lib/wiab` | Images land in `<dir>/images`. |

`wiab_images_url` is marked sensitive and is pushed to the host over SSH rather than through
cloud-init, because cloud-init user-data is readable by anyone with pool access and is stored
rendered in Terraform state.

### The version gate

Fetching is gated on `vm_images_version`, not on file contents. `wiab-deploy` records the last
deployed version in `/etc/wiab/versions` and skips the fetch when it is unchanged.

**So rebuilding an image is not enough.** After a successful CI upload:

1. Bump `vm_images_version` in `terraform.tfvars`.
2. `terraform apply`.

Without the bump, the host keeps the images it already has and the deploy log says
`images: version <n> unchanged, skip`. The transfer itself uses `azcopy --overwrite=ifSourceNewer`,
so a re-fetch only moves files that actually changed.

No backend restart is needed — the VM runtime reads these files when it launches a microVM.

## Verifying

On the host:

```sh
ls -l /var/lib/wiab/images/
cat /etc/wiab/versions
```

You should see the four artifacts owned by `wiab:wiab`, and a recorded `WIAB_IMAGES_VERSION`
matching your tfvars.

Common failures:

- **`images: WIAB_IMAGES_URL unset, skip`** — `wiab_images_url` is empty. Nothing was fetched,
  and Firecracker sandboxes cannot boot.
- **Images never update** — `vm_images_version` was not bumped. See the version gate above.
- **`azcopy` authentication errors** — the SAS token in the URL has expired.
