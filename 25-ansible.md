# Ansible Complete Guide (Beginner to Advanced)
## Configuration Management + Automation for DevOps Engineers

> Practical and detailed guide for local/on-prem/cloud VM automation.
> Covers inventory, ad-hoc commands, playbooks, roles, vault, best practices, troubleshooting.

---

## 1) What is Ansible and Why it Matters

Ansible is an agentless automation tool for:
- server configuration
- application deployment
- patching
- orchestration tasks

## Why use it?
- No agent required on target machines (SSH/WinRM)
- YAML-based playbooks (readable and versionable)
- Idempotent execution (safe repeat runs)
- Strong ecosystem of modules and roles

---

## 2) Install Ansible (Ubuntu)

```bash
sudo apt update
sudo apt install -y ansible
ansible --version
```

(Optional newer version via pip in virtualenv if needed.)

---

## 3) Core Concepts

- **Inventory**: list of managed hosts
- **Ad-hoc command**: one-off task (quick operation)
- **Playbook**: YAML automation workflow
- **Task**: one automation step
- **Role**: reusable automation structure
- **Module**: unit of work (apt, service, copy, user, etc.)
- **Facts**: host system data gathered by Ansible

---

## 4) Inventory Setup

Create `inventory.ini`:

```ini
[web]
web1 ansible_host=10.0.0.11 ansible_user=ubuntu
web2 ansible_host=10.0.0.12 ansible_user=ubuntu

[db]
db1 ansible_host=10.0.0.21 ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

Ping all hosts:
```bash
ansible all -i inventory.ini -m ping
```

---

## 5) Ad-hoc Commands (Quick Ops)

Install package:
```bash
ansible web -i inventory.ini -b -m apt -a "name=nginx state=present update_cache=yes"
```

Check uptime:
```bash
ansible all -i inventory.ini -m shell -a "uptime"
```

Restart service:
```bash
ansible web -i inventory.ini -b -m service -a "name=nginx state=restarted"
```

---

## 6) First Playbook

Create `site.yml`:

```yaml
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Ensure nginx started and enabled
      service:
        name: nginx
        state: started
        enabled: true
```

Run:
```bash
ansible-playbook -i inventory.ini site.yml
```

---

## 7) Variables and Templates

## Variable file
`group_vars/web.yml`
```yaml
nginx_port: 80
app_name: myapp
```

## Jinja2 template
`templates/nginx.conf.j2`
```nginx
server {
    listen {{ nginx_port }};
    server_name _;
    location / {
        return 200 "{{ app_name }} running\n";
    }
}
```

Task using template:
```yaml
- name: Deploy nginx config
  template:
    src: templates/nginx.conf.j2
    dest: /etc/nginx/sites-available/default
  notify: restart nginx
```

Handler:
```yaml
handlers:
  - name: restart nginx
    service:
      name: nginx
      state: restarted
```

---

## 8) Conditionals, Loops, Tags

Conditional:
```yaml
- name: Install curl on Debian
  apt:
    name: curl
    state: present
  when: ansible_os_family == "Debian"
```

Loop:
```yaml
- name: Install common packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - git
    - curl
    - unzip
```

Tags:
```yaml
- name: Install nginx
  apt:
    name: nginx
    state: present
  tags: [nginx,packages]
```

Run specific tag:
```bash
ansible-playbook -i inventory.ini site.yml --tags nginx
```

---

## 9) Roles (Reusable Structure)

Create role:
```bash
ansible-galaxy init roles/nginx
```

Typical role structure:
- `tasks/main.yml`
- `handlers/main.yml`
- `templates/`
- `files/`
- `defaults/main.yml`
- `vars/main.yml`

Use role in playbook:
```yaml
- hosts: web
  become: true
  roles:
    - nginx
```

---

## 10) Ansible Galaxy (Community Roles)

Install role:
```bash
ansible-galaxy role install geerlingguy.nginx
```

Install from requirements file:
`requirements.yml`
```yaml
roles:
  - name: geerlingguy.nginx
```

```bash
ansible-galaxy install -r requirements.yml
```

---

## 11) Vault (Encrypt Secrets)

Create encrypted vars file:
```bash
ansible-vault create group_vars/all/secrets.yml
```

Edit:
```bash
ansible-vault edit group_vars/all/secrets.yml
```

Run playbook with vault prompt:
```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

Or vault password file (CI use carefully):
```bash
ansible-playbook -i inventory.ini site.yml --vault-password-file .vault_pass
```

---

## 12) Dynamic Inventory (Cloud)

For cloud environments (AWS/Azure/GCP), use dynamic inventory plugins.
Benefits:
- auto-discover current instances
- reduce static inventory drift

Example concept:
- `inventory_aws_ec2.yml`
- filter by tags/environment
- run against live fleet

---

## 13) Idempotency and Check Mode

Dry-run:
```bash
ansible-playbook -i inventory.ini site.yml --check
```

Diff mode:
```bash
ansible-playbook -i inventory.ini site.yml --check --diff
```

Why:
- safer change previews
- better CI validation

---

## 14) Common Modules You Should Know

- `apt`, `yum`, `dnf`
- `service`, `systemd`
- `copy`, `template`, `file`, `lineinfile`
- `user`, `group`
- `git`
- `uri`
- `cron`
- `iptables`/firewall modules
- cloud modules (ec2, azure, gcp etc. collections)

---

## 15) CI/CD Integration Pattern

Typical pipeline:
1. `ansible-lint`
2. syntax check
3. `--check` dry run on staging hosts
4. apply on staging
5. approval
6. apply on production

Useful commands:
```bash
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-lint site.yml
```

---

## 16) Troubleshooting

## SSH/auth issues
```bash
ansible all -i inventory.ini -m ping -u ubuntu --private-key ~/.ssh/id_rsa -vvv
```

## Python missing on target
Set interpreter in inventory:
```ini
ansible_python_interpreter=/usr/bin/python3
```

## Permission denied tasks
Use become:
```yaml
become: true
```

## Debug facts
```bash
ansible web1 -i inventory.ini -m setup
```

## Verbose execution
```bash
ansible-playbook -i inventory.ini site.yml -vvv
```

---

## 17) Best Practices

1. Keep playbooks small and role-based  
2. Use group_vars/host_vars cleanly  
3. Encrypt secrets with Ansible Vault  
4. Use tags for controlled runs  
5. Prefer idempotent modules over shell commands  
6. Test with `--check` before prod  
7. Version-control all automation in Git  

---

## 18) Practice Tasks

1. Create inventory with 2 web hosts  
2. Run ad-hoc ping and package install  
3. Write playbook to install/start nginx  
4. Template nginx config from variables  
5. Add handler for restart  
6. Convert playbook logic into reusable role  
7. Encrypt DB password in Vault and consume it  

---

## 19) Daily Cheat Sheet

```bash
ansible all -i inventory.ini -m ping
ansible all -i inventory.ini -m shell -a "uptime"

ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml --check --diff
ansible-playbook -i inventory.ini site.yml --tags nginx

ansible-vault create secret.yml
ansible-vault edit secret.yml
```

---

## Final Notes

- Ansible is one of the fastest ways to standardize server operations.
- Start with inventory + basic playbooks, then mature into roles + vault + CI validation.
- Treat Ansible code like application code: lint, review, test, and version.