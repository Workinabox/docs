# workinabox documentation

## Who each folder is for

Documentation here is split by **whether the reader has the source tree**, not by topic.

> **The test:** would someone running a workinabox box, who will never clone the repo, need this?
>
> If yes, it belongs in `Installation/` or `Configuration/`. If no, it belongs in `Development/`.

A useful second check: if a page cites a repository path, a CI workflow, a Terraform variable or
our own infrastructure, it is developer documentation regardless of what it is about. Deploying
our demo box and building sandbox images are both *operations we perform*, not things a box owner
does.

`Installation/` and `Configuration/` are a **product surface**. Everything in them should make
sense to someone who has a box and nothing else — no repo paths, no crate names, no CI.

| Folder | Audience | Contents |
| --- | --- | --- |
| [`Installation/`](Installation/) | Operators | Standing up a box and connecting third-party services |
| [`Configuration/`](Configuration/) | Operators | Tuning a box that is already running |
| [`Development/`](Development/) | Us | Deploying, building, CI, and working on the code |
| [`obsolete/`](obsolete/) | — | Archive. Read-only, and possibly inaccurate |

## Installation

| Page | What it covers |
| --- | --- |
| [Microsoft Entra](Installation/MICROSOFT_ENTRA.md) | Enterprise SSO — registering the app in a customer's tenant and the `email_verified` requirement |
| [Google OAuth](Installation/GOOGLE_OAUTH.md) | "Continue with Google" — creating the client and how account linking behaves |
| [Email delivery](Installation/EMAIL_DELIVERY.md) | Resend or SMTP, and why unconfigured email fails silently |

## Configuration

| Page | What it covers |
| --- | --- |
| [Configuring your box](Configuration/SETTINGS.md) | Every operator setting, grouped by goal: public URL and TLS, sign-in, email, the first administrator, storage, meetings, logging |

## Development

| Page | What it covers |
| --- | --- |
| [Local development stack](Development/LOCAL_DEVELOPMENT.md) | The compose stack, Mailpit, the mock OIDC provider |
| [Environment variables](Development/ENVIRONMENT_VARIABLES.md) | Complete reference — every variable the system reads, by component, with the file that resolves it |
| [Terraform configuration](Development/TERRAFORM_TFVARS.md) | `terraform.tfvars` — what is required, what is validated, and the constraints validation cannot express |
| [Sandbox VM images](Development/SANDBOX_VM_IMAGES.md) | Building rootfs images in CI and getting them onto a host |

Documentation that lives next to the code it describes, rather than here:

- [`backend/CLAUDE.md`](../backend/CLAUDE.md) — backend conventions
- [`sw-dev-team/docs/ENV.md`](../sw-dev-team/docs/ENV.md) — the agent team's own environment
- [`iac/images/README.md`](../iac/images/README.md) — how sandbox images are built
- [`dev/local/README.md`](../dev/local/README.md) — the local stack reference

## Writing docs here

- **Pick the folder by audience, using the test above.** A page in the wrong folder is worse than
  a missing one, because an operator will act on it.
- **Verify against the code, not against other documents.** Several pages here were written by
  checking the source and found the previous documentation wrong. Cite the file that decides the
  behavior so the next person can re-check it.
- **Don't fork a maintained document.** When something is already documented next to the code,
  link to it and cover only what it doesn't.
- **Nothing in [`obsolete/`](obsolete/) is a source of truth**, and nothing there may be edited.
  See its [README](obsolete/README.md).
