# SYMTEX-CIN Trading Fleet — control panel

A static, single-page dashboard (no backend, no secrets) for non-coders to manage the
trading-VM fleet defined in the **private** repo
[`baid86/symtex-cin-trading-vm`](https://github.com/baid86/symtex-cin-trading-vm).

It lets you:
- **Check / uncheck** which VMs deploy at the scheduled 08:30 IST provision run (toggles a per-VM
  `enabled` flag in `config.yaml`).
- **Add** a new VM (or **Edit** an existing one) via a form.
- **Delete** a VM entry from `config.yaml` (removes it from the fleet so it won't be provisioned;
  this only edits the file — it does not tear down an already-running cloud VM, use Destroy for that).
- **Run the provision / destroy workflows** on demand (via the GitHub Actions API) — no need to open
  the Actions tab.

## Regions
The Add/Edit form's **Region** selector chooses where a VM runs:
- **Central India / UK South / West India** — Azure (writes the region's `azure:` override; Central
  India inherits `common.azure`).
- **Mumbai (AWS)** — Amazon EC2 in `ap-south-1`. Writes `city: Mumbai`, which the provisioner routes
  to AWS (EC2 + security group + Elastic IP + SSM) via `common.cityProviders` + `common.aws` (golden
  AMI). No Azure override is written.

## How it works
The page runs entirely in your browser. It talks to the GitHub Contents API using a **fine-grained
Personal Access Token** you paste once (stored only in your browser's `localStorage`), and it
edits `config.yaml` in the private repo **preserving comments**. Saving creates a normal commit;
the next scheduled GitHub Actions run reads the updated file.

This repo contains **no credentials**. Access to the private fleet repo is entirely controlled by
the token each operator supplies.

## One-time token setup
1. <https://github.com/settings/personal-access-tokens/new>
2. Resource owner `baid86`; Repository access → **Only select repositories** → `symtex-cin-trading-vm`
3. Permissions → Repository → **Contents: Read and write** + **Actions: Read and write** (Actions is
   needed to trigger the provision/destroy workflows from the dashboard)
4. Generate, copy, paste it into the dashboard.
