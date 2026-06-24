# Ansible Docker Swarm

Ansible playbooks and configuration files to bootstrap, configure, and manage Docker Swarm clusters.

## Supported Operating Systems

* Debian

## Features

+ **Python Bootstrapping**: Installs Python and its package manager on target hosts to enable Ansible execution.
+ **Host Configuration**: Automates system hostname configuration and resolves loopback IP mappings.
+ **User and Group Management**:
  - Creates system groups and user accounts.
  - Deploys SSH public keys.
+ **SSH Hardening**: Secures the SSH daemon using a strict, customized configuration file.
+ **Docker Installation**:
  - Installs Docker Engine.
  - Configures user group memberships for non-root Docker execution.
  - Verifies and waits for the Docker service to become active.
+ **Swarm Clustering**:
  - Initializes the Docker Swarm leader node.
  - Registers secondary manager nodes to the swarm.
  - Registers worker nodes to the swarm.

## Installation

### Option 1: Python [venv](https://docs.python.org/3/library/venv.html)

1. Create a virtual environment:

   ```bash
   python -m venv .venv --prompt "ansible-swarm"
   ```

2. Activate the virtual environment:

   ```bash
   source .venv/bin/activate
   ```

3. Install the required dependencies:

   ```bash
   pip install -r requirements.txt
   ```

### Option 2: [uv](https://docs.astral.sh/uv/)

1. Install the dependencies (`uv` creates and manages the virtual environment automatically):

   ```bash
   uv sync
   ```

2. Activate the virtual environment:

   ```bash
   source .venv/bin/activate
   ```

## Configuration

The playbooks require specific variables to be defined in your inventory. Use the following example as a reference structure:

```yaml
all:
  vars:
    ansible_python_interpreter: auto_silent
    ds_core: # Docker Swarm Core variables
      net:
        int:
          cidr: '10.0.10.0/24'
      os:
        groups:
          ssh: &group_ssh sshlogin
          sudo: &group_sudo sudo
          docker: &group_docker docker
        users:
          # This user has SSH access, sudo privileges, and Docker permissions.
          - name: finn
            groups:
              present:
                - *group_ssh
                - *group_sudo
              absent:
            # Generate password using: openssl passwd -6
            password: "$6$3K4J..."
            # Only Ed25519 SSH keys are supported.
            ssh_key: "ssh-ed25519 AAAAC3Nza... finn@advtime"
            docker: true
          # This user has SSH access only, with no sudo or Docker permissions.
          - name: jake
            groups:
              present:
                - *group_ssh
              absent:
            password: "$6$PhYL..."
            ssh_key: "ssh-ed25519 AAAAC3Nza... jake@advtime"

managers:
  hosts:
    vm01:
      ds_host: # Docker Swarm Host variables
        name: master01 # target hostname

workers:
  hosts:
    vm02:
      ds_host:
        name: worker01
```

## Usage

To execute the entire deployment pipeline, run the master playbook:

```bash
ansible-playbook -i path/to/inventory playbooks/master.yaml
```

**Note**: If you place your inventory file inside the `inventory/` directory, Ansible loads it automatically. You can then omit the `-i` argument:

```bash
ansible-playbook playbooks/master.yaml
```

The master playbook executes the following component playbooks sequentially:

1. `python.yaml`: Installs Python on target hosts and configures system hostnames.
2. `users.yaml`: Creates system users, deploys SSH keys, and hardens SSH daemon configuration.
3. `docker.yaml`: Installs Docker Engine and configures user access permissions.
4. `swarm.yaml`: Initializes and bootstraps the Docker Swarm cluster.

## Local Development Lab

You can test these playbooks locally using QEMU virtual machines. A helper guide is provided to walk you through the following steps:

1. Creating multiple Debian virtual machines with QEMU.
2. Building a local network bridge with virtual TAP devices.
3. Configuring a local DHCP server using `dnsmasq` for static IP assignments based on MAC addresses.
4. Configuring passwordless SSH communication between hosts.
5. Tearing down and cleaning up the virtual laboratory environments.

For step-by-step setup instructions, refer to [`cloud_lab_qemu.md`](cloud_lab_qemu.md).

### Managing Docker Swarm from the Host

Since only the SSH port is forwarded from the virtual machines to the host, you must configure SSH port forwarding to manage the Docker Swarm cluster using a local Docker client.

Ensure your local Docker client version is compatible with the server's API version:

```bash
# Docker Client
docker version --format "{{.Client.APIVersion}}"
# Docker Server
ssh vm01 'docker version --format "{{.Server.APIVersion}}"'
```

#### Forwarding the Docker Socket

To manage the cluster, forward the Docker socket to a local TCP port (e.g., `12375`).

```bash
ssh -f -N -L 12375:/var/run/docker.sock vm01
```

Once the tunnel is established, create and switch to a Docker context to route your commands through the tunnel:

```bash
docker context create cloudlab --docker "host=tcp://127.0.0.1:12375"
docker context use cloudlab
docker run hello-world
```

#### Forwarding Additional Services

Any other services running on the Swarm nodes must be forwarded individually to be accessible from the host. For example, to access a web service on port 80:

```bash
ssh -f -N -L 18080:localhost:80 vm01
```

## License

This project is licensed under the GNU Affero General Public License v3 (AGPL-3.0). See [`LICENSE.md`](LICENSE.md) for the full license text.
