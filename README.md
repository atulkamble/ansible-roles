# Ansible Roles: Basic Practice

This hands-on exercise uses your existing Ansible control node at `/etc/ansible` and your Amazon Linux 2023 EC2 hosts.

You will create a reusable `webserver` role that installs Apache (`httpd`), writes a home page from a Jinja2 template, and makes sure Apache is enabled and running. The playbook targets only your `[web]` host, `23.22.227.133`. It does not modify the `[db]` host, `32.202.123.57`.

## 1. Current lab configuration

Your existing inventory:

```ini
[all]
23.22.227.133
32.202.123.57

[web]
23.22.227.133

[db]
32.202.123.57
```

Your `/etc/ansible/ansible.cfg`:

```ini
[defaults]
inventory = ./inventory
remote_user = ec2-user
private_key_file = ./aws/ansible.pem
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false
```

Because `inventory = ./inventory` points to a directory, Ansible reads the `inventory/hosts` file inside it. The key path is relative to `/etc/ansible`. Keep the private key private and do not upload it to GitHub.

## 2. Create role folders

Run these commands on the Ansible control node:

```bash
sudo cd /etc/ansible
sudo mkdir -p roles/webserver/{defaults,tasks,handlers,templates}
```

Expected project layout:

```text
/etc/ansible/
├── ansible.cfg
├── aws/
│   └── ansible.pem
├── inventory/
│   └── hosts
├── roles/
│   └── webserver/
│       ├── defaults/main.yml
│       ├── handlers/main.yml
│       ├── tasks/main.yml
│       └── templates/index.html.j2
└── site.yml
```

## 3. Add default variables

Create `sudo nano roles/webserver/defaults/main.yml`:

```yaml
---
webserver_package: httpd
webserver_service: httpd
webserver_document_root: /var/www/html
webserver_page_title: "Ansible Roles Practice"
webserver_page_message: "This page was deployed using an Ansible role."
```

Defaults provide values the role can use. A playbook or command-line extra variable can override them.

## 4. Add role tasks

Create `sudo nano roles/webserver/tasks/main.yml`:

```yaml
---
- name: Install Apache HTTP Server
  ansible.builtin.dnf:
    name: "{{ webserver_package }}"
    state: present

- name: Deploy the homepage
  ansible.builtin.template:
    src: index.html.j2
    dest: "{{ webserver_document_root }}/index.html"
    owner: root
    group: root
    mode: "0644"
  notify: Restart Apache

- name: Ensure Apache is enabled and running
  ansible.builtin.service:
    name: "{{ webserver_service }}"
    state: started
    enabled: true
```

Amazon Linux 2023 uses DNF. Its Apache package and service are both named `httpd`.

## 5. Add a handler

Create `sudo nano roles/webserver/handlers/main.yml`:

```yaml
---
- name: Restart Apache
  ansible.builtin.service:
    name: "{{ webserver_service }}"
    state: restarted
```

The homepage task notifies this handler. Ansible restarts Apache only when the template changes.

## 6. Add the Jinja2 page template

Create `sudo nano roles/webserver/templates/index.html.j2`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>{{ webserver_page_title }}</title>
  </head>
  <body>
    <h1>{{ webserver_page_title }}</h1>
    <p>{{ webserver_page_message }}</p>
    <p>Server: {{ inventory_hostname }}</p>
  </body>
</html>
```

Ansible substitutes the variables when it copies the template to the target host.

## 7. Create the playbook

Create `sudo nano /etc/ansible/playbooks/site.yml`:

```yaml
---
- name: Configure Apache on the web server
  hosts: web
  roles:
    - webserver
```

# apply playbook yaml 
```
cd /etc/ansible
sudo ansible-playbook -i inventory/hosts playbooks/site.yml
```

The `roles` section tells Ansible to load tasks and supporting files from `roles/webserver/`. Since privilege escalation is already enabled in `ansible.cfg`, the playbook does not need a separate `become` line.

## 8. Check SSH and inventory

Make sure the key exists and has restrictive permissions:

```bash
cd /etc/ansible
chmod 400 aws/ansible.pem
ansible-inventory --graph
ansible web -m ansible.builtin.ping
```

Expected ping result includes `"ping": "pong"`. If SSH fails, confirm the public IP, key path, key permissions, security-group SSH access, and `ec2-user` login.

## 9. Preview and apply

Preview the changes without applying them:

```bash
ansible-playbook site.yml --check --diff
```

Apply the role:

```bash
ansible-playbook site.yml
```

Run it a second time:

```bash
ansible-playbook site.yml
```

The second run should report mostly `ok`, because the host already matches the requested configuration. This is Ansible idempotency.

## 10. Verify Apache and the page

Run on the target through Ansible:

```bash
ansible web -a "systemctl is-active httpd"
ansible web -a "curl -s http://localhost"
```

To access the page from your control node, allow inbound TCP port 80 in the web EC2 instance's security group:

```bash
curl http://23.22.227.133
```

## Practice challenges

1. Change the default page message in `roles/webserver/defaults/main.yml` and apply the playbook again.
2. Override the message for one run without editing files:
   ```bash
   ansible-playbook site.yml -e 'webserver_page_message=Hello from my role'
   ```
3. Add `webserver_page_color` to defaults and use it as the page background color in the template.
4. Create a second role named `common` that installs `git`.
5. Add another host to the `[web]` group and observe how Ansible applies the same role to each host.
6. Change `hosts: web` to `hosts: all` and explain why that would install Apache on the database host too.

## Key terms

- **Role:** a reusable structure for tasks, variables, handlers, templates, and other files.
- **Defaults:** baseline role variables that can be overridden.
- **Task:** one operation Ansible performs on managed hosts.
- **Template:** a file containing Jinja2 expressions that Ansible renders with variables.
- **Handler:** a task that runs when notified, commonly used to restart or reload a service after a configuration change.
- **Idempotency:** repeated runs maintain the desired state without making unnecessary changes.

## Troubleshooting

| Symptom | Check |
|---|---|
| `UNREACHABLE` or public key denied | Confirm the key path is `/etc/ansible/aws/ansible.pem`, mode is `400`, inventory IP is current, and SSH user is `ec2-user`. |
| SSH timeout | Confirm inbound TCP 22 is allowed from the control node and the instance is reachable. |
| sudo failure | Confirm `ec2-user` has sudo privileges and `become` settings are active. |
| HTTP timeout from your machine | Allow inbound TCP 80 in the web server's security group. |
| Package or service not found | This role targets Amazon Linux 2023 and expects package/service name `httpd`. |

