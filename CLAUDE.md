# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

Malignia is a small collection of Ansible playbooks and supporting config used to provision and firewall a single home server (referred to as host `malignia`, inventory IP `127.0.0.1` — these playbooks are run locally on the target host via `connection: local`, not over SSH to a remote inventory). There is no application code, build system, or test suite; everything here is infrastructure-as-code for one machine.

## Commands

- Run a playbook: `ansible-playbook -i inventory.txt <playbook>.yml`
- Playbooks in `test-playbooks/` use their own local `test-playbooks/inventory.txt`.
- There is no lint/test tooling configured; validate playbook syntax with `ansible-playbook --syntax-check -i inventory.txt <playbook>.yml` before relying on a change.

## Architecture

- `docker_services.yml` — discovers `docker-*` directories under `./docker_sources` and brings each up via `community.docker.docker_compose_v2`. Expects a `docker_sources/` directory (not checked into this repo) containing one subfolder per service.
- `set_iptables_rules.yml` + `vars/iptables.yml` — creates a `trusted-allow` iptables chain, jumps `DOCKER-USER` and `INPUT` traffic on `fwd_dest_ports`/`input_dest_ports` into it, allows traffic from `trusted_addrs` (private ranges + specific public IPs), and drops everything else on that chain.
- `set_tv_allowed_nets.yml` + `set_iptables_tv_allowed.yml` + `vars/tvheadend.yml` — builds a separate `tv_allowed` chain restricting TVHeadend ports (9981/9982) to networks resolved at runtime via `whois -h whois.radb.net` lookups against the AS numbers in `tv_allowed_as`, plus the static CIDRs in `tv_allowed_net`. `set_tv_allowed_nets.yml` is the entrypoint; it loops over `asnet` and includes `set_iptables_tv_allowed.yml` per AS to populate the chain before logging/dropping the rest.
- These two iptables playbooks are additive/order-dependent: they insert rules and flush/create chains rather than being fully idempotent from a clean box — rerunning after manual iptables changes can produce a different rule order than a fresh run.
- `vars/*.yml` centralize the tunables (trusted addresses, ports, chain names, allowed ASes/nets) referenced by the playbooks above via Jinja `{{ }}` interpolation — change values there rather than inline in the playbooks.

## Working conventions

- Playbooks assume they execute on the target host itself (`connection: local`, `hosts: 127.0.0.1` or `all`); they are not written to be run against a remote inventory.
- `docker_services.yml` depends on a `docker_sources/` layout that isn't part of this repo — treat it as an external prerequisite, not something to scaffold here.
- iptables playbooks are destructive/stateful (`chain_management`, `flush: yes`); test rule changes in `vars/iptables.yml` / `vars/tvheadend.yml` before running against a live firewall.
