# Ansible Roles Practice

Beginner practice for reusable Ansible roles using the existing `/etc/ansible` control-node layout and Amazon Linux 2023 targets.

This exercise applies the `webserver` role to the host in your `[web]` group. It installs Apache HTTP Server (`httpd`), deploys a templated homepage, and ensures Apache starts at boot. Your `[db]` host is not modified.

## Repository contents

```text
ansible.cfg.example
inventory/hosts.example
site.yml
roles/webserver/
├── defaults/main.yml
├── handlers/main.yml
├── tasks/main.yml
└── templates/index.html.j2
```

## Lab prerequisites

Use your existing inventory and SSH key from `/etc/ansible`. The example inventory reflects the lab addresses shared for this exercise, but adjust them if your public IPs have changed. Never commit your private key.

Your configuration should be:

```ini
[defaults]
inventory = ./inventory
remote_user = ec2-user
private_key_file = ./aws/ansible.pem
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

## Install the role into the current lab

From a clone of this repository, copy its files into your existing project. For example, if cloned under `/tmp/ansible-roles`:

```bash
sudo mkdir -p /etc/ansible/roles
sudo cp -R /tmp/ansible-roles/roles/webserver /etc/ansible/roles/
sudo cp /tmp/ansible-roles/site.yml /etc/ansible/site.yml
cd /etc/ansible
chmod 400 aws/ansible.pem
```

Keep your existing `ansible.cfg` and `inventory/hosts`; the example files are references.

## Run and verify

```bash
ansible-inventory --graph
ansible web -m ansible.builtin.ping
ansible-playbook site.yml --check --diff
ansible-playbook site.yml
ansible web -a "systemctl is-active httpd"
ansible web -a "curl -s http://localhost"
```

To view the site remotely, allow inbound TCP port 80 in the web instance's security group, then run:

```bash
curl http://23.22.227.133
```

## Role concepts

- **Role:** reusable directory of tasks, defaults, handlers, and templates.
- **Defaults:** low-priority variables that can be overridden.
- **Tasks:** ordered operations run against matched hosts.
- **Template:** Jinja2 file rendered with variables and host details.
- **Handler:** runs when notified by a changed task; here Apache restarts when the homepage changes.
- **Idempotency:** repeated runs converge on the desired state; unchanged resources should report `ok`.

## Practice challenges

1. Override the message without editing the role:
   ```bash
   ansible-playbook site.yml -e 'webserver_page_message=Hello from my role'
   ```
2. Add a `webserver_page_color` default and use it in the HTML template.
3. Create a `common` role that installs `git`.
4. Change `hosts: web` to `hosts: all` only after considering that this also installs Apache on the database server.
5. Run the playbook twice and explain the difference between `changed` and `ok`.

## Troubleshooting

- **SSH/public key failure:** check the key path, mode `400`, current inventory IP, and `ec2-user` remote user.
- **sudo failure:** confirm the EC2 user can run sudo.
- **HTTP timeout:** allow inbound TCP port 80 in the web server security group.
- **Package/service mismatch:** this role targets Amazon Linux 2023 and uses DNF with the `httpd` package and service.
