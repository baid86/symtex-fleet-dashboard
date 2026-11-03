# SYMTEX-CIN Trading Fleet — control panel

A static, single-page dashboard (no backend, no secrets) for non-coders to manage the
trading-VM fleet defined in the **private** repo
[`baid86/symtex-cin-trading-vm`](https://github.com/baid86/symtex-cin-trading-vm).

## Layout
- **Trading-day strip** (top): the current IST time, what the market is doing, and a 08:00–16:30
  bar marking 08:30 auto start, 09:15 open, 15:30 close and 16:00 auto delete. Each auto marker
  shows *paused* when that workflow's `schedule:` cron is commented out (read live from the
  workflow file). The strip also holds the fleet-wide **Start N VMs** (every VM set to deploy) and
  **Delete all N** buttons, plus the last few workflow runs with their status, refreshed every 20 s
  while one is queued or running.
- **VMs** list: search, an All / Deploying / Off filter, and sortable columns. Each row has:
  - a **Deploy** switch (the per-VM `enabled` flag in `config.yaml`; off = skipped by provisioning);
  - **▶ Start** — provision just that VM now. If Deploy is off it is turned on first (the
    provisioner skips `enabled: false` VMs);
  - **■ Delete** — deprovision just that VM now: deletes the cloud VM, disk and NIC, keeps its public
    IP for reuse. The `config.yaml` entry stays;
  - **⋯** menu with **Edit** and **Remove** — Remove deprovisions the VM (a no-op if it's already
    gone) **and** deletes its entry from `config.yaml`. The destroy run is triggered first, so it
    still sees the entry's region routing.
- **Add VM** / **Edit** open a side panel. A new VM is committed to `config.yaml` straight away;
  edits stay pending until **Save changes**, so several can go in one commit. VM names must be unique.
- **Kite IDs**: how many VMs use each Kite login (set to deploy, out of total).
- **Bottom bar**: the latest message, plus **Unsaved changes**, **Discard** and **Save changes**.

Every Start / Delete / Remove first saves any unsaved changes, so the workflow runs against the
config you see; if that save fails (e.g. a 409 conflict) the workflow is not triggered. The **Dry
run** switch in the header applies to all of these (a dry-run Remove keeps the entry). Every action
is confirmed in a dialog that says exactly what will happen; destructive ones default to Cancel.

Per-VM deprovision needs the `only` input on `destroy-vm.yml` in the fleet repo (maps to
`Remove-TradingVm.ps1 -Name`).

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
