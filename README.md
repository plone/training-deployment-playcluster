# DevOps Ansible Operations for playcluster 🚀

Welcome to the DevOps documentation for playcluster! In this guide, we'll walk you through the setup and deployment process, ensuring a smooth and efficient development workflow. We leverage the power of [Ansible](https://www.ansible.com/), [Docker](https://www.docker.com/), and [Docker Swarm](https://docs.docker.com/engine/swarm/) to automate, containerize, and orchestrate application deployment. 🛠️🐳🌐

- **Ansible** empowers us to automate tasks like software provisioning, configuration management, and application deployment. It's like having a robot assistant that takes care of the repetitive tasks, freeing you to focus on more strategic activities! 🤖✨

- **Docker** encapsulates our application and its dependencies into a container to ensure consistency across multiple development, testing, and deployment environments. It's like packing your entire application, including the environment it runs in, into a portable box that you can run anywhere! 📦🚀

- **Docker Swarm** takes it a step further by turning a group of Docker engines into a single, virtual Docker engine. It allows us to deploy our containers across multiple machines, enhancing availability and scalability. It's like having a swarm of bees working harmoniously to build, run, and scale your application! 🐝🌟

### Our Docker Stack 📚

We deploy a robust website running [Plone](https://plone.org/) using a Docker stack that consists of:

- **Traefik:** Serves as the router and SSL termination, integrated with [Let's Encrypt](https://letsencrypt.org/) for free SSL certificates, ensuring that our website is secure and trusted. 🔒🌐

- **Plone Frontend using Volto:** A modern, fast, React-based frontend that delivers an exceptional user experience. It's like having a sleek, high-performance car to navigate the web! 🏎️💨

- **Plone Backend:** Responsible for the API, it's the engine under the hood, ensuring that data is processed, stored, and retrieved efficiently. 🏭🚀

- **Postgres 18 Database:** A reliable, robust database to store the site data, ensuring that our content is safe, secure, and quickly accessible. 🗃️⚡


Now, let’s dive into the setup! 🏊‍♂️💫
## Setup

Ensure you navigate to the `devops/ansible/` folder before executing any commands listed in this document. From the root of your repository, execute:

```shell
cd devops/ansible/
```

### Environment Configuration

Start by creating an `.env` file in the `devops/ansible/` folder. You can copy the existing `.env_dist` file as a starting point:

```shell
cp .env_dist .env
```

Edit the `.env` file to suit your environment. For example:

```
ANSIBLE_REMOTE_PORT=22
DEPLOY_ENV=prod
DEPLOY_HOST=play1.playcluster.plone.org
DEPLOY_PORT=22
DEPLOY_USER=plone
DOCKER_CONFIG=.docker
STACK_NAME=playcluster-plone-org
```

Note: The `.env` file is included in `.gitignore`, ensuring environment-specific configurations aren't pushed to the repository.


### Server installation

You need either a Ubuntu or Debian based system for each playcluster node, enable SSH, and install a supported version of Python 3 on that system.


### Ansible Installation

Execute the following to create a Python 3 virtual environment and install Ansible along with its dependencies:

```shell
make install
```

### Inventory Configuration

Modify `devops/ansible/inventory/hosts.yml` with the appropriate connection details:

```yaml
---
cluster:
  hosts:
    play1.playcluster.plone.org:
      ansible_user: root
      ansible_host: play1.playcluster.plone.org
      host: play1
      hostname: play1.playcluster.plone.org
      swarm_node:
        labels:
          type: manager
          env: production

```

## Server Setup

With the correct information in `devops/ansible/inventory/hosts.yml`, test the connection to the server with:

```shell
uv run ansible-playbook playbooks/_connect.yml
```

And then, if the connection is successful, initiate the remote server setup by running:

```shell
uv run ansible-playbook playbooks/setup.yml
```

This command executes the Ansible playbook `devops/playbooks/setup.yml` on the remote server, performing various setup tasks including user creation, SSH setup, Docker installation, and more.

## Container Registry on play4 📦

`play4` is the standalone host: it runs the GitLab CI runners and, at
`https://registry.playcluster.plone.org`, a [zot](https://zotregistry.dev/) container registry
behind its own Traefik.

**It is deliberately not a swarm node.** The swarm lives on play1–3, and joining play4 to it would
put a second Traefik in the cluster competing for ports 80/443 and drop the CI runner's containers
into an orchestrated environment. So the registry is a plain `docker compose` project, deployed by
Ansible in the same shape the swarm stacks use:

| | Swarm (play1–3) | Compose (play4) |
| --- | --- | --- |
| Definition | `stacks` in `inventory/group_vars/all/stacks.yml` | `compose_projects` in `inventory/group_vars/standalone/compose.yml` |
| Deployment | `tasks/stacks/task_deploy.yml` | `tasks/compose/task_deploy.yml` |
| Files | `etc/stacks/*.yml` | `etc/compose/*.yml` |

The registry's own configuration — accounts, ACLs and retention — lives in
`inventory/group_vars/standalone/registry.yml`.

### Deploying

Deploy or redeploy just the registry:

```shell
uv run ansible-playbook playbooks/setup_ci.yml --limit standalone --tags registry
```

Everything lands in `/srv/registry` on the host as `compose.yml` plus a generated `.env`, so the
usual commands work there directly:

```shell
ssh root@play4.playcluster.plone.org 'cd /srv/registry && docker compose ps && docker compose logs --tail=50 registry'
```

### Accounts

Two machine accounts, with genuinely different privileges:

| Account | Rights | Used by |
| --- | --- | --- |
| `ci` | read, create, update, delete | GitLab CI build jobs pushing images and the BuildKit layer cache |
| `deploy` | read only | The swarm, pulling during `docker stack deploy` |

These are registry logins, not accounts on the hosts. The SSH account that runs `docker stack
deploy` is `plone` (`users.default`); `root` (`users.setup`) is only for provisioning.

The split is not cosmetic. `docker stack deploy --with-registry-auth` copies the deploying client's
credentials onto every swarm node, where they persist so nodes can re-pull after a reboot. Push
rights must never land there.

Anonymous access is denied. Passwords live in the vault and are turned into a bcrypt `htpasswd` file
on the host; read them back with:

```shell
uv run ansible-vault view inventory/group_vars/all/vault.yml
```

They map onto the `playcluster-demo` pipeline's CI/CD variables as `REGISTRY_USER` /
`REGISTRY_PASSWORD` (from `ci`) and `REGISTRY_PULL_USER` / `REGISTRY_PULL_PASSWORD` (from `deploy`),
alongside `REGISTRY_IMAGE_PREFIX=registry.playcluster.plone.org/playcluster-demo`.

### Web UI

zot ships its own web UI (zui) on port **7443**:

<https://registry.playcluster.plone.org:7443/>

It is not a second container. The `ghcr.io/project-zot/zot` image already has the extensions
compiled in — the startup log reports a `binary-type` of `…-search-sync-ui-userprefs` — so it is
enabled purely by `registry.ui` in `inventory/group_vars/standalone/registry.yml`.

Sign in with the registry accounts themselves, `ci` or `deploy`; there is no separate UI login and
no Traefik basic auth in front, which would only mean a second password prompt.

**Why a separate port.** zot serves the UI and the registry API from one listener, so enabling the
UI also puts it on 443 at `/`. To keep 443 an API-only endpoint, the 443 router is narrowed to
`PathPrefix(/v2/)` — the whole OCI distribution API, and all a docker client ever uses — while the
UI gets its own entrypoint. Turning the UI off restores the unrestricted 443 rule automatically.

Anonymous visitors to 7443 get the static page shell and `/v2/_zot/ext/mgmt`, which zui reads to
discover the login methods; it returns the zot version and `{"htpasswd":{}}`, no usernames. Every
data path — GraphQL search, `/v2/_catalog`, manifests — returns 401 without credentials. The one
tradeoff is that the exact zot version is publicly readable on that port.

CVE scanning is deliberately **off**. It is the expensive half of the UI: zot downloads Trivy's
vulnerability databases into `_trivy/` under the storage root and rescans on a timer, on a host that
is also running the CI builds. Set `registry.ui.cve_scanning: true` to enable it and watch memory.

### Traefik dashboard

The Traefik in front of the registry exposes its admin UI on port **8443**:

<https://registry.playcluster.plone.org:8443/dashboard/> — the trailing slash matters.

Log in as `admin` with the same password as the cluster Traefik UI; both read
`vault.traefik.ui_basic_auth`. The credential is mounted as a `usersfile` rather than set in a
`basicauth.users` label, because the `$apr1$` hash contains `$` (which docker compose would
interpolate) and labels are readable by anything that can reach the docker socket — on play4 that
includes CI job containers.

It answers on the registry's own hostname so it reuses that Let's Encrypt certificate instead of
requesting a second one.

**This is a public admin UI on a host where `ufw` is inactive.** Turn it off when you are done:
set `traefik_dashboard.enabled: false` in `inventory/group_vars/standalone/registry.yml` and
redeploy — the entrypoint, published port, mount and labels all disappear together.

### Retention

zot prunes on a schedule so the `*/cache` repositories do not grow without bound — BuildKit's
`mode=max` cache turns over on every build.

It ships with **`dryRun: true`**, meaning zot only logs what it *would* delete. Confirm the policies
match what you expect before letting it delete anything:

```shell
ssh root@play4.playcluster.plone.org 'cd /srv/registry && docker compose logs registry | grep -i retention'
```

Then set `retention.dryRun: false` in `inventory/group_vars/standalone/registry.yml` and redeploy.
