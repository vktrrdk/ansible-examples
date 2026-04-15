# BibiGrid/Slurm Cluster Setup with Ansible

This repository contains Ansible playbooks to automate the setup of a BiBiGrid/Slurm cluster with examples, that are typical for the use with R.

## Overview

The setup consists of the following playbooks:

1. **`create_users.yml`** - Creates user accounts on all worker nodes with SSH keys
2. **`install_software.yml`** - Installs R on all nodes (master and workers)
3. **`remove_users.yml`** - Removes users from worker nodes (keeps home directories)

These are imported by **`site.yml`** which serves as the main entry point.

---

## Quick Start

```bash
# Run the complete cluster setup
ansible-playbook site.yml -i inventory.yml

# Run only user creation
ansible-playbook create_users.yml -i inventory.yml

# Run only software installation
ansible-playbook install_software.yml -i inventory.yml

# Remove users (keeps home directories with data)
ansible-playbook remove_users.yml -i inventory.yml

# Run with verbose output
ansible-playbook site.yml -i inventory.yml -v

# Check mode (dry run - show what would change)
ansible-playbook site.yml -i inventory.yml --check

# Limit to specific hosts
ansible-playbook site.yml -i inventory.yml --limit master
ansible-playbook site.yml -i inventory.yml --limit workers
```

---

## How the Files Work

### `inventory.yml` - Your Cluster Definition

The inventory file defines which hosts belong to which groups:

```yaml
all:
  children:
    workers:
      hosts:
        bibigrid-worker-n8h3jsn3ohtpu84-[0:1]  # Worker nodes (range syntax)
    master:
      hosts:
        localhost  # The master node (running Ansible from)
```

**Key concepts:**
- `hosts:` defines individual servers or use patterns like `[0:1]` for ranges
- `children:` creates groups within the `all` group
- You can run plays against specific groups (e.g., `hosts: workers`)

---

### `vars/` Directory - Configuration Files

The `vars/` directory contains YAML files with variables used by playbooks. This separates **configuration** from **logic**.

#### `vars/users.yml` - User Configuration

Defines users to create and users to remove on worker nodes:

```yaml
users:
  - name: alice
    groups: sudo
    shell: /bin/bash
    ssh_key: ecdsa-sha2-nistp256 AAAAE2VjZHNh...

remove_users:
  - formercoworker
  - olduser
```

**How it works:**
- The `create_users.yml` playbook loads this file via `vars_files`
- It iterates over the `users` list using `loop: "{{ users }}"`
- Each user's properties (name, groups, shell, ssh_key) are accessed via `{{ item.name }}`, etc.

**To customize:**
- Add/remove users in the `users` list. Each user gets a home directory and their SSH key authorized
- Add users to `remove_users` list to remove them from the system (their home directories and data are preserved)

#### `vars/software.yml` - Software Configuration

Defines R packages and system dependencies for the cluster:

```yaml
r_packages:
  - r-base
  - r-base-dev

apt_dependencies:
  - build-essential
  - gfortran
  - libcurl4-openssl-dev
  - libssl-dev
  - libxml2-dev
  - libicu-dev
  - libfontconfig1-dev
  - libcairo2-dev
  - libxt-dev
  - zlib1g-dev
  - libbz2-dev
  - liblzma-dev
```

**How it works:**
- Loaded by `install_software.yml` via `vars_files: - vars/software.yml`
- R packages are installed via `apt`
- System dependencies include libraries needed for building R packages from CRAN

**To customize:**
- Add/remove R packages in the `r_packages` list (e.g., `r-cran-dplyr`, `r-cran-ggplot2`)
- Add system dependencies when R packages fail to install (error messages indicate missing `-dev` packages)

---

## Playbook Structure Explained

### `site.yml` - Main Entry Point

```yaml
---
- import_playbook: create_users.yml
- import_playbook: install_software.yml
```

**Type:** *Import Playbook* - Includes other playbooks

- `import_playbook:` includes another playbook file
- Plays are executed in order (users first, then software)
- This is the simplest way to combine multiple playbooks

**Note:** The `remove_users.yml` playbook is commented out in `site.yml` by default. Run it separately when needed to remove users.

### `create_users.yml` - User Management Playbook

```yaml
- name: Create users on all workers
  hosts: workers          # Run on the 'workers' group from inventory
  become: true            # Run as sudo/root
  vars_files:
    - vars/users.yml      # Load user configuration

  tasks:
    - name: Create users
      ansible.builtin.user:    # Ansible's built-in user module
        name: "{{ item.name }}"
        state: present
        shell: "{{ item.shell }}"
        groups: "{{ item.groups }}"
        create_home: yes
      loop: "{{ users }}"     # Iterate over users list

    - name: Add SSH public keys
      ansible.posix.authorized_key:   # Posix module for SSH keys
        user: "{{ item.name }}"
        state: present
        key: "{{ item.ssh_key }}"
      loop: "{{ users }}"
```

**Task Types Used:**

| Task | Module | Purpose |
|------|--------|---------|
| User creation | `ansible.builtin.user` | Creates system users with specified properties |
| SSH key setup | `ansible.posix.authorized_key` | Adds SSH public keys to user's `~/.ssh/authorized_keys` |

**Key Concepts:**
- `become: true` elevates privileges to root (required for user management)
- `loop:` iterates over list items (like a for-loop)
- `{{ }}` syntax inserts variable values
- `vars_files:` loads external YAML files with variables

### `install_software.yml` - Software Installation Playbook

This playbook has **two separate plays** targeting different host groups.

---

### `remove_users.yml` - Remove Users Playbook

Removes users from worker nodes while preserving their home directories:

```yaml
- name: Remove users from all workers
  hosts: workers
  become: true
  vars_files:
    - vars/users.yml

  tasks:
    - name: Remove users (keep home directories)
      ansible.builtin.user:
        name: "{{ item }}"
        state: absent
        remove: no      # Do NOT remove home directory
        force: no       # Do NOT force removal if process is running
      loop: "{{ remove_users }}"
```

**Task Types Used:**

| Task | Module | Purpose |
|------|--------|---------|
| User removal | `ansible.builtin.user` | Removes user accounts, preserves home directories |

**Key Parameters:**
- `state: absent` - Ensures the user does not exist
- `remove: no` - **Crucial:** Preserves the home directory with all data
- `force: no` - Only removes user if not actively logged in

**Why preserve home directories?**
- User data (R scripts, analysis results, configuration) is kept
- Only the account is removed, not the files
- Data can be archived or reassigned later if needed

#### Play 1: Master Node

```yaml
- name: Install R on master node
  hosts: master           # Targets the 'master' group (localhost)
  become: true
  vars_files:
    - vars/software.yml

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install system dependencies
      apt:
        name: "{{ apt_dependencies }}"
        state: present

    - name: Install R packages
      apt:
        name: "{{ r_packages }}"
        state: present
```

**Task Types Used:**

| Task | Module | Purpose |
|------|--------|---------|
| Package update | `apt` | Updates apt cache before installs |
| Package install | `apt` | Installs packages from Debian repos |

#### Play 2: Worker Nodes

```yaml
- name: Install R on all worker nodes
  hosts: workers          # Targets the 'workers' group
  become: true
  vars_files:
    - vars/software.yml

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install system dependencies
      apt:
        name: "{{ apt_dependencies }}"
        state: present

    - name: Install R packages
      apt:
        name: "{{ r_packages }}"
        state: present

    - name: Create R library directory for all users
      file:
        path: "/home/{{ item.name }}/R/x86_64-linux-gnu-library/{{ ansible_date_time.epoch | int | string[:4] }}"
        state: directory
        owner: "{{ item.name }}"
        group: "{{ item.name | groupby(item.groups.split(',')[0]) | first }}"
        mode: '0755'
      loop: "{{ users }}"   # Re-uses users list from vars/users.yml
      when: users is defined
```

---

## Ansible Modules Reference

### Built-in Modules (No Extra Installation)

| Module | Purpose | Docs |
|--------|---------|------|
| `ansible.builtin.user` | Manage user accounts | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html) |
| `ansible.builtin.file` | Manage files/directories | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html) |
| `ansible.builtin.get_url` | Download files | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html) |
| `ansible.builtin.copy` | Copy files to remote hosts | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html) |
| `ansible.builtin.template` | Render Jinja2 templates | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html) |

### APT Modules (Debian/Ubuntu)

| Module | Purpose | Docs |
|--------|---------|------|
| `apt` | Manage packages | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_module.html) |
| `apt_key` | Manage APT keys | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_key_module.html) |
| `apt_repository` | Add/remove repos | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_repository_module.html) |

### System Modules

| Module | Purpose | Docs |
|--------|---------|------|
| `systemd` | Manage systemd services | [Link](https://docs.ansible.com/projects/ansible/latest/collections/ansible/builtin/systemd_service_module.html#ansible-collections-ansible-builtin-systemd-service-module) |
| `service` | Generic service management | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html) |
| `cron` | Manage cron jobs | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/cron_module.html) |

### POSIX/Remote Modules

| Module | Purpose | Docs |
|--------|---------|------|
| `ansible.posix.authorized_key` | Manage SSH keys | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/posix/authorized_key_module.html) |
| `ansible.posix.synchronize` | rsync file sync | [Link](https://docs.ansible.com/ansible/latest/collections/ansible/posix/synchronize_module.html) |

### Network Modules

| Module | Purpose | Docs |
|--------|---------|------|
| `ufw` | UFW firewall rules | [Link](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html) |
| `firewalld` | firewalld rules | [Link](https://docs.ansible.com/ansible/latest/collections/community/general/firewalld_module.html) |

---

## Variables and Filters

### Variable Syntax

```yaml
# Access variable
{{ variable_name }}

# Access list item
{{ item.name }}

# Access nested variable
{{ rstudio.version }}

# Default value if undefined
{{ some_var | default('default_value') }}

# String formatting
"Hello {{ name | default('Guest') }}"
```

### Useful Filters

| Filter | Purpose | Example |
|--------|---------|---------|
| `default()` | Provide default value | `{{ var \| default('fallback') }}` |
| `bool` | Convert to boolean | `{{ 'true' \| bool }}` |
| `upper/lower` | Case conversion | `{{ name \| upper }}` |
| `length` | Get length | `{{ list \| length }}` |
| `join` | Join list items | `{{ list \| join(',') }}` |

---

## Common Patterns

### Idempotency

Ansible tasks should be **idempotent** - running them multiple times should produce the same result:

```yaml
# GOOD: state: present ensures package is installed (or already is)
- name: Install package
  apt:
    name: vim
    state: present

# BAD: Command that always runs
- name: Install package
  command: apt install vim
```

### Error Handling

```yaml
- name: Install RStudio
  block:
    - name: Download
      get_url:
        url: "https://example.com/rstudio.deb"
        dest: /tmp/rstudio.deb
    - name: Install
      apt:
        deb: /tmp/rstudio.deb
  rescue:
    - name: Cleanup on failure
      file:
        path: /tmp/rstudio.deb
        state: absent
      changed_when: false
```

### Tags for Selective Execution

```yaml
tasks:
  - name: Quick task
    debug:
      msg: "quick"
    tags:
      - quick

  - name: Slow task
    debug:
      msg: "slow"
    tags:
      - slow

# Run only specific tags:
# ansible-playbook site.yml --tags "quick,slow"
# ansible-playbook site.yml --skip-tags "slow"
```

---

## Troubleshooting

```bash
# Check what would change (dry run)
ansible-playbook site.yml --check

# Show what matches your hosts
ansible all --list-hosts

# Test connection to all hosts
ansible all -m ping

# Run single task for debugging
ansible master -m apt -a "name=vim state=present"

# See full debug output
ansible-playbook site.yml -vvvv
```

---

## File Structure

```
.
├── site.yml                 # Main entry point - imports other playbooks
├── create_users.yml         # Creates user accounts on workers
├── install_software.yml     # Installs R and dependencies on all nodes
├── remove_users.yml         # Removes users (keeps home directories)
├── inventory.yml            # Defines cluster hosts and groups
├── vars/
│   ├── users.yml           # User definitions (name, groups, ssh_key)
│   └── software.yml        # Software configuration (packages, versions)
└── README.md               # This file
```

---

## Customizing for Your Cluster

1. **Update `inventory.yml`** - Replace hostnames with your actual worker/master nodes
2. **Update `vars/users.yml`** - Add your users with their SSH keys
3. **Update `vars/software.yml`** - Add/remove R packages as needed
4. **Run the playbook** - `ansible-playbook site.yml -i inventory.yml`

For large clusters, consider:
- Using `serial: 1` to run on one host at a time
- Adding `--limit` to target specific hosts
- Using `--tags` to run specific parts of the playbook
