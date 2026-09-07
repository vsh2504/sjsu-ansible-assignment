# Ansible Web Server Deployment and Lifecycle Management

This repository contains an Ansible configuration that deploys and removes an
Nginx web server across two local Ubuntu virtual machines: `vm1` and `vm2`.

Each web server listens on TCP port `8080` and serves instance-specific content:

- **VM1:** `Hello World from SJSU-1`
- **VM2:** `Hello World from SJSU-2`

The project was developed on macOS using Multipass for virtualization and
Ansible for configuration management.

## Project Structure

```text
sjsu-ansible-assignment/
├── .gitignore
├── ansible.cfg
├── hosts.ini
├── README.md
├── webserver.yml
└── templates/
    └── default.conf.j2
```

The project-local `.ssh/` directory is intentionally excluded from version
control because it contains the SSH private key used to connect to the VMs.

## Architecture

The Mac is the Ansible control node. Ansible connects to both Ubuntu VMs over
SSH and applies the same playbook to each host. The `sjsu_id` inventory variable
generates the correct message for each VM without duplicating tasks.

```text
macOS control node
├── SSH → vm1 → Nginx :8080 → Hello World from SJSU-1
└── SSH → vm2 → Nginx :8080 → Hello World from SJSU-2
```

## Prerequisites

- macOS
- [Homebrew](https://brew.sh/)
- [Canonical Multipass](https://canonical.com/multipass)
- [Ansible](https://docs.ansible.com/)
- Git

Install Multipass and Ansible with Homebrew:

```bash
brew install multipass ansible
```

## 1. Provision the Virtual Machines

Create two Ubuntu 22.04 LTS virtual machines:

```bash
multipass launch 22.04 --name vm1
multipass launch 22.04 --name vm2
multipass list
```

Record the IPv4 address shown for each VM. These addresses are required in
`hosts.ini` and in the browser or `curl` verification commands.

If `22.04` is unavailable, inspect the images visible to Multipass and use the
matching Ubuntu 22.04 alias, such as `jammy`:

```bash
multipass find
```

## 2. Configure SSH Access

Create a project-local SSH key from the repository root:

```bash
mkdir -p .ssh
ssh-keygen -t rsa -b 4096 -N "" -f .ssh/id_rsa
chmod 600 .ssh/id_rsa
```

Authorize the public key on both VMs:

```bash
multipass exec vm1 -- bash -c "echo '$(cat .ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys"
multipass exec vm2 -- bash -c "echo '$(cat .ssh/id_rsa.pub)' >> ~/.ssh/authorized_keys"
```

> **Security:** Only the public key is copied to the VMs. Never commit
> `.ssh/id_rsa` or any other private key to GitHub.

## 3. Configure the Ansible Inventory

Replace `<VM1_IP>` and `<VM2_IP>` in `hosts.ini` with the addresses returned by
`multipass list`:

```ini
[webservers]
vm1 ansible_host=<VM1_IP> ansible_user=ubuntu sjsu_id=1
vm2 ansible_host=<VM2_IP> ansible_user=ubuntu sjsu_id=2

[webservers:vars]
ansible_ssh_private_key_file=.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

The project-level `ansible.cfg` supplies the inventory and avoids an interactive
host-key prompt for these disposable local VMs:

```ini
[defaults]
inventory = hosts.ini
host_key_checking = False
interpreter_python = auto_silent
retry_files_enabled = False
```

> **Production note:** Disabling host-key checking is appropriate only for this
> isolated local lab. Long-lived environments should verify fingerprints and
> manage trusted entries in `known_hosts`.

## 4. Validate the Configuration

Check the playbook syntax:

```bash
ansible-playbook --syntax-check webserver.yml
```

Verify that Ansible can reach both VMs:

```bash
ansible all -m ping
```

Both hosts should return `SUCCESS` and `"ping": "pong"`.

If the command reports `Host key verification failed`, confirm that
`ansible.cfg` exists in the repository root and rerun the command from that
directory.

## 5. Deploy the Web Servers

Run only the deployment play:

```bash
ansible-playbook hosts.ini webserver.yml --tags deploy
```

The deployment performs the following work on both VMs:

1. Installs Nginx with `apt`.
2. Deploys `templates/default.conf.j2` as the default Nginx site.
3. Configures Nginx to listen on TCP port `8080`.
4. Creates `/var/www/html/index.html` using the host's `sjsu_id` value.
5. Validates the Nginx configuration with `nginx -t`.
6. Starts Nginx and enables it at boot.

A successful play recap should show `failed=0` and `unreachable=0` for both
hosts.

## 6. Verify the Deployment

Test both web servers from the Mac:

```bash
curl http://<VM1_IP>:8080
# Hello World from SJSU-1

curl http://<VM2_IP>:8080
# Hello World from SJSU-2
```

The pages can also be opened in a browser:

- `http://<VM1_IP>:8080`
- `http://<VM2_IP>:8080`

The browser address bar should show port `8080`, and each page should display
the message assigned to that VM.

## 7. Verify Idempotency

Run the deployment command a second time:

```bash
ansible-playbook hosts.ini webserver.yml --tags deploy
```

The second execution should complete successfully without unexpected changes.
Normally, the play recap reports `changed=0` for both hosts.

## 8. Undeploy the Web Servers

Run only the teardown play:

```bash
ansible-playbook hosts.ini webserver.yml --tags undeploy
```

The undeploy play:

1. Stops and disables Nginx.
2. Purges the `nginx` and `nginx-common` packages.
3. Removes orphaned package dependencies.
4. Deletes `/var/www/html` and the generated page.

A successful play recap should again show `failed=0` and `unreachable=0` for
both hosts.

## 9. Verify Teardown

Confirm that neither VM is serving HTTP traffic on port `8080`:

```bash
curl --connect-timeout 5 http://<VM1_IP>:8080
curl --connect-timeout 5 http://<VM2_IP>:8080
```

Both requests should fail with a connection-refused or failed-to-connect
message. This is the expected negative test after undeployment.

## VM Cleanup

```bash
multipass delete vm1 vm2
multipass purge
```

This final step removes the virtual machines themselves. It is separate from
the Ansible undeploy play, which removes only the web server resources.

