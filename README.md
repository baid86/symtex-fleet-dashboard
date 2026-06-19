# SYMTEX-CIN Trading Fleet — control panel

A static, single-page dashboard (no backend, no secrets) for non-coders to manage the
trading-VM fleet defined in the **private** repo
[`baid86/symtex-cin-trading-vm`](https://github.com/baid86/symtex-cin-trading-vm).

It lets you:
- **Check / uncheck** which VMs deploy at the scheduled 08:30 IST provision run (toggles a per-VM
  `enabled` flag in `config.yaml`).
- **Add a new VM** via a form.

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
3. Permissions → Repository → **Contents: Read and write**
4. Generate, copy, paste it into the dashboard.
