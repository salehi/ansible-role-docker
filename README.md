# Ansible Role: Docker

[![CI](https://github.com/salehi/ansible-role-docker/actions/workflows/ci.yml/badge.svg)](https://github.com/salehi/ansible-role-docker/actions/workflows/ci.yml)

An Ansible Role that installs [Docker](https://www.docker.com) on Linux.

> **Note:** This is a fork of [geerlingguy.docker](https://github.com/geerlingguy/ansible-role-docker) that adds a `docker_proxy_env` variable, so the role can install Docker through an HTTP/SOCKS proxy (useful where `download.docker.com` is geo-blocked). All credit for the original role goes to Jeff Geerling. Published to Galaxy as `salehi.docker`.

## Requirements

None.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```yaml
# Edition can be one of: 'ce' (Community Edition) or 'ee' (Enterprise Edition).
docker_edition: 'ce'
docker_packages:
  - "docker-{{ docker_edition }}"
  - "docker-{{ docker_edition }}-cli"
  - "docker-{{ docker_edition }}-rootless-extras"
  - "containerd.io"
  - docker-buildx-plugin
docker_packages_state: present
```

The `docker_edition` should be either `ce` (Community Edition) or `ee` (Enterprise Edition). 
You can also specify a specific version of Docker to install using the distribution-specific format: 
Red Hat/CentOS: `docker-{{ docker_edition }}-<VERSION>` (Note: you have to add this to all packages);
Debian/Ubuntu: `docker-{{ docker_edition }}=<VERSION>` (Note: you have to add this to all packages).

You can control whether the package is installed, uninstalled, or at the latest version by setting `docker_packages_state` to `present`, `absent`, or `latest`, respectively. Note that the Docker daemon will be automatically restarted if the Docker package is updated. This is a side effect of flushing all handlers (running any of the handlers that have been notified by this and any other role up to this point in the play).

```yaml
docker_obsolete_packages:
  - docker
  - docker.io
  - docker-engine
  - docker-doc
  - docker-compose
  - docker-compose-v2
  - podman-docker
  - containerd
  - runc
```

`docker_obsolete_packages` for different os-family:

- [`RedHat.yaml`](./vars/RedHat.yml)
- [`Debian.yaml`](./vars/Debian.yml)
- [`Suse.yaml`](./vars/Suse.yml)

A list of packages to be uninstalled prior to running this role. See [Docker's installation instructions](https://docs.docker.com/engine/install/debian/#uninstall-old-versions) for an up-to-date list of old packages that should be removed.

```yaml
docker_service_manage: true
docker_service_state: started
docker_service_enabled: true
docker_service_start_command: ""
docker_restart_handler_state: restarted
```

Variables to control the state of the `docker` service, and whether it should start on boot. If you're installing Docker inside a Docker container without systemd or sysvinit, you should set `docker_service_manage` to `false`.

```yaml
docker_install_compose_plugin: true
docker_compose_package: docker-compose-plugin
docker_compose_package_state: present
```

Docker Compose Plugin installation options. These differ from the below in that docker-compose is installed as a docker plugin (and used with `docker compose`) instead of a standalone binary.

```yaml
docker_install_compose: false
docker_compose_version: "v2.32.1"
docker_compose_arch: "{{ ansible_facts.architecture }}"
docker_compose_url: "https://github.com/docker/compose/releases/download/{{ docker_compose_version }}/docker-compose-linux-{{ docker_compose_arch }}"
docker_compose_path: /usr/local/bin/docker-compose
```

Docker Compose installation options. Set `docker_compose_version` to a specific tag (e.g. `v2.32.1`) to pin a release, or to `latest` to query the GitHub API at run time and install the most recent release.

```yaml
docker_add_repo: true
```

Controls whether this role will add the official Docker repository. Set to `false` if you want to use the default docker packages for your system or manage the package repository on your own.

```yaml
docker_repo_url: https://download.docker.com/linux
```

The main Docker repo URL, common between Debian and RHEL systems.

```yaml
docker_apt_release_channel: stable
docker_apt_ansible_distribution: "{{ 'ubuntu' if ansible_facts.distribution in ['Pop!_OS', 'Linux Mint'] else ansible_facts.distribution }}"
docker_apt_repo_url: "{{ docker_repo_url }}/{{ docker_apt_ansible_distribution | lower }}"
docker_apt_gpg_key: "{{ docker_repo_url }}/{{ docker_apt_ansible_distribution | lower }}/gpg"
docker_apt_filename: "docker"
```

(Used only for Debian/Ubuntu.) You can switch the channel to `nightly` if you want to use the Nightly release.

`docker_apt_ansible_distribution` is a workaround for Ubuntu variants which can't be identified as such by Ansible, and is only necessary until Docker officially supports them.

You can change `docker_apt_repo_url` if you need to point Debian or Ubuntu systems at an internal mirror or cache.
You can change `docker_apt_gpg_key` to a different url if you are behind a firewall or provide a trustworthy mirror.
`docker_apt_filename` controls the name of the source list file created in `sources.list.d`. If you are upgrading from an older (<7.0.0) version of this role, you should change this to the name of the existing file (e.g. `download_docker_com_linux_debian` on Debian) to avoid conflicting lists.

```yaml
docker_yum_repo_url: "{{ docker_repo_url }}/{{ 'fedora' if ansible_facts.distribution == 'Fedora' else 'rhel' if ansible_facts.distribution == 'RedHat' else 'centos' }}/docker-{{ docker_edition }}.repo"
docker_yum_repo_enable_test: '0'
docker_yum_gpg_key: "{{ docker_repo_url }}/{{ 'fedora' if ansible_facts.distribution == 'Fedora' else 'rhel' if ansible_facts.distribution == 'RedHat' else 'centos' }}/gpg"
```

(Used only for RedHat/CentOS.) You can enable the Test repo by setting the respective vars to `1`.

You can change `docker_yum_gpg_key` to a different url if you are behind a firewall or provide a trustworthy mirror.
Usually in combination with changing `docker_yum_repository` as well.

```yaml
docker_users: []
```

A list of system users to be added to the `docker` group (so they can use Docker on the server). Example:

```yaml
docker_users:
  - user1
  - user2
```

```yaml
docker_daemon_options: {}
```

Custom `dockerd` options can be configured through this dictionary representing the json file `/etc/docker/daemon.json`. Example:

```yaml
docker_daemon_options:
  storage-driver: "overlay2"
  log-opts:
    max-size: "100m"
```

```yaml
docker_proxy_env: {}
```

Proxy environment variables to inject into every task that makes an outbound network call (package installation, GPG key fetch, repo setup, docker-compose binary download). When left empty (the default), behavior is identical to upstream — no regression.

Accepts any standard proxy keys (`http_proxy`, `https_proxy`, `no_proxy`, `all_proxy`, etc.) which are passed through as-is to each task's environment. Example:

```yaml
- hosts: docker
  roles:
    - role: salehi.docker
      vars:
        docker_proxy_env:
          https_proxy: "http://user:pass@proxy.example.com:7777"
          no_proxy: "localhost,127.0.0.1,mirror.local"
```

You can also use a SOCKS proxy via `all_proxy` instead of HTTP proxies — but do not set both `all_proxy` and `http_proxy`/`https_proxy` at the same time, as tools handle the combination inconsistently:

```yaml
docker_proxy_env:
  all_proxy: "socks5://proxy.example.com:1080"
  no_proxy: "localhost,127.0.0.1"
```

```yaml
docker_egress_ip_check: true
docker_egress_ip_check_url: "https://ifconfig.io/ip"
```

Before the OS-specific setup tasks run, the role queries `docker_egress_ip_check_url` and prints the host's public egress IP. This makes it easy to confirm which IP (or proxy) is used to reach the Docker package repositories — handy alongside `docker_proxy_env`. The request honours `docker_proxy_env`, and a failure (no connectivity, blocked endpoint) is ignored so it never aborts the install.

The endpoint is fully configurable; any service that returns the IP as plain text works, e.g. `https://ifconfig.me/ip`, `https://api.ipify.org`, or `https://icanhazip.com`. Set `docker_egress_ip_check: false` to skip the outbound request entirely.

## Use with Ansible (and `docker` Python library)

Many users of this role wish to also use Ansible to then _build_ Docker images and manage Docker containers on the server where Docker is installed. In this case, you can easily add in the `docker` Python library using the `geerlingguy.pip` role:

```yaml
- hosts: all

  vars:
    pip_install_packages:
      - name: docker

  roles:
    - geerlingguy.pip
    - salehi.docker
```

## Dependencies

None.

## Example Playbook

```yaml
- hosts: all
  roles:
    - salehi.docker
```

## License

MIT / BSD

## Author Information

This role was created in 2017 by [Jeff Geerling](https://www.jeffgeerling.com/), author of [Ansible for DevOps](https://www.ansiblefordevops.com/).
