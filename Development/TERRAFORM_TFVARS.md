# Terraform configuration (`terraform.tfvars`)

What must be set before a `terraform apply`, which variables are validated, and the constraints
that are not expressible as validation.

Copy `iac/terraform.tfvars.example` to `iac/terraform.tfvars` and fill it in. Never commit
`terraform.tfvars`.

For the environment variables these ultimately produce on the host, see
[`ENVIRONMENT_VARIABLES.md`](ENVIRONMENT_VARIABLES.md).

## Required — `plan` fails without them

| Variable | What it is |
| --- | --- |
| `xoa_url` | Xen Orchestra endpoint, e.g. `wss://xoa.lan`. |
| `xoa_token` | XO API token. Sensitive. |
| `template_name` | VM template name, must match your XO. |
| `network_name` | Network name, must match your XO. |
| `storage_repository` | Storage repository name, must match your XO. |
| `host_ip` | Static LAN IP for the VM, outside the DHCP pool. |
| `gateway` | Network gateway. |
| `ssh_authorized_key` | Public key installed on the host. |
| `domain` | FQDN the box serves. |
| `letsencrypt_email` | Contact address for certificate issuance. |
| `announced_address` | Address WebRTC announces to clients — the public WAN IP when served through NAT, otherwise `host_ip`. |
| `db_password` | Local Postgres password. Sensitive, and validated — see below. |

Everything else has a default.

## Validated variables

These fail `plan` with an explanatory message rather than breaking at apply time or, worse,
silently at runtime.

### `db_password`

**At least 16 characters from `[A-Za-z0-9._~-]`.** No default.

The charset is deliberate, not fussiness: the value is interpolated into a `DATABASE_URL` and
into a shell-sourced env file, so URL and shell metacharacters would break one or both. Generate
one out of band:

```sh
openssl rand -base64 24 | tr -d '/+=' | cut -c1-32
```

There is no default because a shared, memorable password ships to every deployment that forgets
to set one.

> **Rotating an existing box.** Changing this in tfvars updates what Terraform configures, but
> the running Postgres keeps whatever password it already has until the DB provisioning re-runs.
> If your box predates this validation it may still be on the old default — rotate it
> deliberately.

### `ssh_admin_cidr`

Empty, or a valid CIDR. Default is empty.

**Empty means SSH accepts connections from any source.** That is only appropriate when port 22 is
genuinely unreachable from the internet — and note that the example `announced_address` is a
public IP, which implies it is not. Provisioning logs a warning when this is unset.

### `rtc_max_port`

Must be above `rtc_min_port`. Defaults are `40000`–`40999`.

## Constraints validation cannot express

### The media port range must match the backend

The firewall opens exactly `rtc_min_port`–`rtc_max_port`, and the backend pins its WebRTC
transport range from `WIAB_MEDIASOUP_MIN_PORT`/`WIAB_MEDIASOUP_MAX_PORT`. Provisioning writes the
backend's values from these variables, so they agree by construction — but only if both land
together.

**If you deploy the backend and the firewall rule separately, the backend change must be live
before or with the narrowed firewall rule.** A mismatch presents as media that negotiates
successfully and then carries no audio, which is an unpleasant thing to debug.

Two transports per peer, so the default range carries roughly 500 concurrent peers.

### Version-gated fetches

`vm_images_version` and `db_provision_version` are cache keys, not descriptions. Bump them to make
the corresponding work re-run on an existing host:

- `vm_images_version` — re-fetches sandbox VM images after a CI rebuild. See
  [`SANDBOX_VM_IMAGES.md`](SANDBOX_VM_IMAGES.md).
- `db_provision_version` — re-applies Postgres provisioning over SSH without rebuilding the VM.

Similarly, `backend_version`/`frontend_version` pinned to `latest` will not be detected as a
change. Pin explicit tags to drive updates, or force it with
`apply -replace=null_resource.deploy_app`.

### Auth enables itself from credentials

There are no enable flags in tfvars. Google login turns on when `google_client_id` **and**
`google_client_secret` are both non-empty; enterprise SSO turns on when `oidc_issuer`,
`oidc_client_id` and `oidc_client_secret` are all non-empty. Terraform derives
`WIAB_AUTH_GOOGLE_ENABLED`/`WIAB_AUTH_OIDC_ENABLED` from that.

Setup runbooks: [`../Installation/GOOGLE_OAUTH.md`](../Installation/GOOGLE_OAUTH.md),
[`../Installation/MICROSOFT_ENTRA.md`](../Installation/MICROSOFT_ENTRA.md).

### `xoa_insecure`

Defaults to `"false"`. Setting it to `"true"` disables TLS verification to Xen Orchestra. If your
tfvars carries `"true"`, the fix is a real certificate on XO — not a configuration change here.

## Secrets

`xoa_token`, `db_password`, `resend_api_key`, `google_client_secret`, `oidc_client_secret`,
`wiab_models_url` and `wiab_images_url` are marked sensitive. The last two are sensitive because
the URL embeds a SAS token.

Sensitive values are pushed to the host **over SSH after boot**, not through cloud-init, because
cloud-init user-data is retrievable by anyone with pool access and is stored rendered in
Terraform state. They land in `/etc/wiab/wiab.env`, created `0640 root:wiab` before anything is
written to it.

Terraform state itself still contains these values. Treat the state file as a secret.

## Check an existing `terraform.tfvars`

`iac/terraform.tfvars.example` carried two defects until recently. A `terraform.tfvars` copied
from it before then still carries them — neither is corrected by updating the example.

1. **`db_password` was `"wiab"`**, which is 4 characters against a 16 minimum. If yours still has
   it, `plan` now fails; generate a real one (see above) and rotate the running database, which
   keeps its existing password regardless of what tfvars says.
2. **`vm_image_files` omitted `initramfs`**, listing only
   `["base.ext4", "developer.ext4", "vmlinux"]`. The initramfs is what assembles the overlay
   root, so a host configured from the old example fetches images that cannot boot a sandbox.
   Add it and bump `vm_images_version` to trigger the fetch.
