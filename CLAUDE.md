# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible role (`qualys_cloud_agent`) for deploying Qualys Cloud Agent across Linux, macOS, Windows, and Unix (AIX, Solaris, BSD) using the Qualys Cloud Agent binary download API. Published to Ansible Galaxy under the `nelssec` namespace.

## Testing and Validation

There is no automated test suite. To validate changes locally:

```bash
# Syntax check
ansible-playbook examples/install.yml --syntax-check

# Dry run against a target (requires inventory and credentials)
ansible-playbook examples/install.yml --check --diff -i inventory

# Run with specific platform tags
ansible-playbook playbook.yml --tags linux
ansible-playbook playbook.yml --tags macos
ansible-playbook playbook.yml --tags windows
```

## Architecture

### Task Flow

`tasks/main.yml` orchestrates everything in this order:
1. **Input validation** — asserts required variables exist and have safe formats (SSRF/injection prevention on URLs and IDs)
2. **OS variable loading** — `vars/` files selected via `first_found` lookup: `{Distribution}-{MajorVersion}.yml` → `{Distribution}.yml` → `{os_family}.yml` → `default.yml`
3. **Platform-specific install** — dispatches to `install_linux.yml`, `install_macos.yml`, or `install_windows.yml` based on `ansible_facts['system']`/`ansible_facts['os_family']`
4. **Service management** — `service.yml` (skipped when `qualys_agent_state == "absent"`)
5. **Post-install validation** — `validate.yml` (controlled by `qualys_agent_validate_install`)

### Download Mechanism (`tasks/download.yml`)

The role auto-detects `qualys_platform_type` and `qualys_architecture` from Ansible facts, then constructs an XML POST request to the Qualys API at `/qps/rest/1.0/download/ca/downloadbinary/`. Every OS has parallel task blocks (Linux/macOS/Unix vs Windows) because Windows uses `ansible.windows.*` modules.

### Key Patterns

- **Dual-path tasks**: Most tasks have separate Linux/macOS and Windows blocks due to different Ansible module requirements (`ansible.builtin.*` vs `ansible.windows.*`).
- **Handlers use `listen`**: Handlers respond to topic names (`restart qualys agent`, `reload qualys agent`) rather than specific handler names, so both Linux and Windows handlers trigger from the same notify.
- **`no_log: true`** on all tasks that handle credentials or API calls.
- **`qualys_agent_state`**: Controls install (`present`/`latest`) vs removal (`absent`). Removal logic is in each platform's install file, gated early with `meta: end_host`.

### Required Variables (no defaults — must be provided)

`qualys_activation_id`, `qualys_customer_id`, `qualys_api_username`, `qualys_api_password`, `qualys_cloud_platform`

## Collections Dependencies

`ansible.windows`, `ansible.posix`, `community.general` (declared in `meta/main.yml`).
