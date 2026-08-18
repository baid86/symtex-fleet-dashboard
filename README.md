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

## Order brokers
The Add/Edit form's **Order broker** dropdown picks how the VM's trading app places orders:

- **MetaTrader 5** (default) — the classic flow: writes the per-VM `mt5:` block
  (login / password / server) and the provisioner performs the one-time MT5 terminal login.
- **Any other broker** (SU Exch, P3 Exch, Builcon Pro, GM Global, Anchor Alpha, Ocean Exch,
  Money Plant, PlatformAPI) — an HTTP broker the app selects via the `ORDER_BROKER` env var.
  The form swaps to that broker's fields (username/account + password, plus a server/company
  field where the broker has one, prefilled with its usual value: `su`, `p3`, `UNICKON`), and
  writes them as env vars (`env.ORDER_BROKER`, `env.<PREFIX>_USERNAME`, `env.<PREFIX>_PASSWORD`,
  …) instead of an `mt5:` block. With no `mt5:` block, the provisioner skips MT5 entirely
  (no terminal launch, no login wait) and just verifies the app came up.

Each non-MT5 broker also gets an optional **Order lots** field (`<PREFIX>_ORDER_LOTS`, or
`_ORDER_AMOUNT` for SU/P3 Exch; GM Global accepts a number or `max`) — blank omits the key so the
app sizes orders itself (notional / broker breakup lot). PlatformAPI additionally offers
**Max lots per order** (`PLATFORMAPI_MAX_ORDER_LOTS`), a hard per-order ceiling applied after
sizing — keep it ≥ Order lots or it silently clamps every order down.

Switching a VM's broker on Edit scrubs the previous broker's credential keys from the entry, so
no stale logins linger in `config.yaml`. The broker list lives in one registry (`BROKERS`) at the
top of `index.html` — adding a broker there (label + env key names) is the only change needed.

The table's **Login** and **Broker / server** columns show the MT5 login/server for MT5 VMs, and
the broker's username / `broker · server` for the rest.

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
