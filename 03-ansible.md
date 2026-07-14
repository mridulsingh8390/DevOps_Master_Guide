# Ansible Complete Guide (Beginner to Advanced)
## For DevOps Engineers (Ubuntu 24.04, local/on-prem)

> This file is detailed and practical.
> Each section includes:
> - What/Why
> - Commands
> - Explanation
> - Verification
> - Common mistakes and fixes
> - Practice tasks

---

## Table of Contents

1. What is Ansible and why it matters  
2. Installation and version checks  
3. Inventory (INI + YAML)  
4. Ad-hoc commands (module-based operations)  
5. Playbooks (core structure)  
6. Variables, facts, conditionals, loops  
7. Templates and handlers  
8. Roles and Ansible Galaxy  
9. Vault (secrets management)  
10. Tags, limits, check mode, diff mode  
11. Ansible configuration (`ansible.cfg`)  
12. Common modules by category  
13. Troubleshooting and debugging  
14. Best practices  
15. Practice lab tasks  
16. Daily cheat sheet

---

## 1) What is Ansible and Why it Matters

## What
Ansible is an agentless automation tool for configuration management, provisioning, and orchestration.

## Why
- No agent required on managed hosts (SSH/WinRM based)
- Human-readable YAML playbooks
- Idempotent operations (safe re-run)
- Good for server setup, app deployment, patching, user mgmt, etc.

---

## 2) Installation and Version Checks

## 2.1 Install Ansible
```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

## 2.2 Verify Python on remote hosts
```bash
ansible all -i inventory.ini -m setup -a "filter=ansible_python*"
```

---

## 3) Inventory (INI + YAML)

## 3.1 INI inventory example (`inventory.ini`)
```ini
[web]
192.168.1.10 ansible_user=ubuntu
192.168.1.11 ansible_user=ubuntu

[db]
192.168.1.20 ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

## 3.2 YAML inventory example (`inventory.yml`)
```yaml
all:
  vars:
    ansible_python_interpreter: /usr/bin/python3
  children:
    web:
      hosts:
        web1:
          ansible_host: 192.168.1.10
          ansible_user: ubuntu
    db:
      hosts:
        db1:
          ansible_host: 192.168.1.20
          ansible_user: ubuntu
```

## 3.3 Validate inventory
```bash
ansible-inventory -i inventory.ini --list
ansible-inventory -i inventory.ini --graph
```

---

## 4) Ad-hoc Commands (Quick Operations)

## 4.1 Connectivity check
```bash
ansible all -i inventory.ini -m ping
```

## 4.2 Run shell/command
```bash
ansible web -i inventory.ini -a "uptime"
ansible all -i inventory.ini -m shell -a "df -h"
ansible all -i inventory.ini -m command -a "whoami"
```

## 4.3 Package/service management
```bash
ansible all -i inventory.ini -m apt -a "name=nginx state=present update_cache=yes" --become
ansible all -i inventory.ini -m service -a "name=nginx state=started enabled=yes" --become
```

## 4.4 File/user operations
```bash
ansible all -i inventory.ini -m file -a "path=/tmp/demo state=directory mode=0755"
ansible all -i inventory.ini -m copy -a "src=/etc/hosts dest=/tmp/hosts.backup"
ansible all -i inventory.ini -m user -a "name=devops state=present groups=sudo append=yes" --become
```

## Notes
- Use `--become` for privileged tasks.
- Prefer modules over raw shell commands for idempotency.

---

## 5) Playbooks (Core Structure)

## 5.1 Basic playbook (`site.yml`)
```yaml
- name: Configure web servers
  hosts: web
  become: yes
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Ensure nginx started
      service:
        name: nginx
        state: started
        enabled: yes
```

## 5.2 Run playbook
```bash
ansible-playbook -i inventory.ini site.yml
```

## 5.3 Verify
```bash
ansible web -i inventory.ini -a "systemctl status nginx --no-pager" --become
```

---

## 6) Variables, Facts, Conditionals, Loops

## 6.1 Variables in playbook
```yaml
- hosts: web
  vars:
    package_name: nginx
  tasks:
    - name: Install package using variable
      apt:
        name: "{{ package_name }}"
        state: present
      become: yes
```

## 6.2 Facts
Gathered automatically (unless disabled).
Use:
```yaml
- debug:
    msg: "OS = {{ ansible_distribution }} {{ ansible_distribution_version }}"
```

## 6.3 Conditionals
```yaml
- name: Install nginx only on Debian family
  apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"
  become: yes
```

## 6.4 Loops
```yaml
- name: Install multiple packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - vim
  become: yes
```

---

## 7) Templates and Handlers

## 7.1 Jinja2 template (`templates/nginx.conf.j2`)
```nginx
server {
    listen {{ nginx_port }};
    server_name _;
    location / {
        return 200 "Hello from {{ inventory_hostname }}\n";
    }
}
```

## 7.2 Use template + handler
```yaml
- hosts: web
  become: yes
  vars:
    nginx_port: 80
  tasks:
    - name: Deploy nginx config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: restart nginx

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

Why handlers:
- Trigger restart only when config actually changes.

---

## 8) Roles and Ansible Galaxy

## 8.1 Create role skeleton
```bash
ansible-galaxy init roles/nginx
tree roles/nginx
```

Standard structure:
- defaults/
- vars/
- tasks/
- handlers/
- templates/
- files/
- meta/

## 8.2 Role-based playbook
```yaml
- hosts: web
  become: yes
  roles:
    - nginx
```

## 8.3 Install role from Galaxy
```bash
ansible-galaxy install geerlingguy.nginx
ansible-galaxy list
```

---

## 9) Vault (Secrets Management)

## 9.1 Create and edit vault file
```bash
ansible-vault create group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
ansible-vault view group_vars/all/vault.yml
```

## 9.2 Encrypt/decrypt existing file
```bash
ansible-vault encrypt secrets.yml
ansible-vault decrypt secrets.yml
ansible-vault rekey secrets.yml
```

## 9.3 Run with vault password
```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

Or with password file:
```bash
ansible-playbook -i inventory.ini site.yml --vault-password-file .vault_pass.txt
```

## Security note
Never commit `.vault_pass.txt` to Git.

---

## 10) Tags, Limits, Check Mode, Diff Mode

## 10.1 Tags
```yaml
- name: Install nginx
  tags: [install, nginx]
  apt:
    name: nginx
    state: present
  become: yes
```

Run by tag:
```bash
ansible-playbook -i inventory.ini site.yml --tags nginx
ansible-playbook -i inventory.ini site.yml --skip-tags nginx
```

## 10.2 Limit hosts
```bash
ansible-playbook -i inventory.ini site.yml --limit web
```

## 10.3 Dry run and diffs
```bash
ansible-playbook -i inventory.ini site.yml --check --diff
```

Why:
- `--check`: simulate changes
- `--diff`: show file-level differences

---

## 11) ansible.cfg

## Sample `ansible.cfg`
```ini
[defaults]
inventory = ./inventory.ini
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
timeout = 30

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False
```

Why:
- Reduces repetitive CLI flags
- Sets team-wide behavior

---

## 12) Common Modules by Category

## Package/service
- `apt`, `yum`, `dnf`, `package`
- `service`, `systemd`

## Files/content
- `copy`, `template`, `file`, `lineinfile`, `blockinfile`, `replace`

## Users/groups
- `user`, `group`, `authorized_key`

## Commands
- `command`, `shell`, `script`
(Prefer module-based tasks when available)

## Archive/download
- `get_url`, `unarchive`

## Facts/debug
- `setup`, `debug`, `assert`, `fail`

---

## 13) Troubleshooting and Debugging

## 13.1 Increase verbosity
```bash
ansible-playbook -i inventory.ini site.yml -vvv
```

## 13.2 SSH troubleshooting
```bash
ssh ubuntu@192.168.1.10
```

## 13.3 Common issues

### Issue: UNREACHABLE
- Wrong IP/user/key/firewall
- Fix SSH connectivity first

### Issue: sudo password prompt
- Add `--ask-become-pass` or configure passwordless sudo

### Issue: Python missing on target
- Install Python on target machine first

### Issue: YAML syntax error
- Validate indentation (spaces, no tabs)

### Useful checks
```bash
ansible-inventory -i inventory.ini --graph
ansible-config dump --only-changed
```

---

## 14) Best Practices

1. Keep playbooks idempotent
2. Prefer modules over raw shell
3. Use roles for reusable logic
4. Store secrets in Vault
5. Use `--check --diff` before production
6. Use tags for selective execution
7. Keep inventory clean and grouped logically
8. Version-control playbooks and roles

---

## 15) Practice Lab Tasks

1. Create inventory for 2 web + 1 db host  
2. Run ping and uptime ad-hoc commands  
3. Install nginx via playbook  
4. Deploy templated nginx config and trigger handler  
5. Convert playbook into role  
6. Move secret variable to Vault  
7. Run same playbook using `--check --diff`  
8. Debug one intentionally broken task using `-vvv`

---

## 16) Daily Ansible Cheat Sheet

```bash
# Connectivity
ansible all -i inventory.ini -m ping

# Run ad-hoc command
ansible web -i inventory.ini -a "uptime"

# Run playbook
ansible-playbook -i inventory.ini site.yml

# Dry run
ansible-playbook -i inventory.ini site.yml --check --diff

# Verbose debug
ansible-playbook -i inventory.ini site.yml -vvv
```

---

## 17) Dynamic Inventory

## Why
Static inventory files don't scale for cloud environments where hosts come and go. Dynamic inventory queries the source of truth (AWS, Azure, GCP, etc.) at runtime.

## Example: AWS EC2 dynamic inventory plugin (`aws_ec2.yml`)
```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
```

Requires collection:
```bash
ansible-galaxy collection install amazon.aws
```

Use it:
```bash
ansible-inventory -i aws_ec2.yml --graph
ansible-playbook -i aws_ec2.yml site.yml
```

---

## 18) ansible-pull (Agent-Style Model)

## Why
Instead of pushing from a control node, each host pulls its own config from Git on a schedule — useful for large fleets or disconnected/edge nodes.

```bash
ansible-pull -U https://github.com/<org>/<repo>.git site.yml
```

Typical setup: cron job or systemd timer running `ansible-pull` every N minutes on each managed node.

---

## 19) Collections

## Why
Ansible ships core modules separately from community/vendor content via **collections** — the modern way to extend Ansible (replaces older "role from Galaxy" pattern for provider-specific modules).

```bash
ansible-galaxy collection install community.general
ansible-galaxy collection install azure.azcollection
ansible-galaxy collection list
```

Reference in playbook:
```yaml
- hosts: web
  tasks:
    - name: Use a community module
      community.general.ufw:
        rule: allow
        port: '22'
      become: yes
```

`requirements.yml` for reproducible installs:
```yaml
collections:
  - name: community.general
    version: ">=8.0.0"
  - name: amazon.aws
```
```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Final Notes

- Start with simple playbooks, then refactor into roles.
- Vault is mandatory for any secrets.
- Most production incidents come from missing validation—use `--check --diff`.
- For cloud fleets, prefer dynamic inventory and pin collection versions in `requirements.yml`.
