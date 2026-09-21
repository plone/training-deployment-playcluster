# training-deployment-playcluster

Ansible that turns four bare Ubuntu servers into the **playcluster**: a Docker Swarm with Traefik
and Portainer, plus a separate host running a GitLab Runner and a private container registry.

This is the *cluster* half of the Plone deployment training. Deploying Plone sites onto the cluster
happens in the companion repository, **training-deployment-gitlabdeploy**: a cookieplone project whose
GitLab pipeline builds its images into the cluster's registry and deploys it to
`playcluster.plone.org`. Its pipeline runs on GitLab.com, at
[gitlab.com/plone-training1/training-deployment-gitlabdeploy](https://gitlab.com/plone-training1/training-deployment-gitlabdeploy),
and it is mirrored to
[github.com/plone/training-deployment-gitlabdeploy](https://github.com/plone/training-deployment-gitlabdeploy).

## What you get

| Host | Role |
| --- | --- |
| `play1` | Swarm manager: Traefik, Portainer, and the deployed sites |
| `play2` | Swarm worker |
| `play3` | Swarm worker, labelled for persistent storage |
| `play4` | **Not** in the swarm: GitLab Runner and the [zot](https://zotregistry.dev/) container registry |

Everything runs from your own machine. Ansible reaches the servers over SSH as `root` and needs
nothing installed on them beforehand.

## Where to find what

Once the cluster runs, it has five web addresses on two hosts:

| Address | What | Login |
| --- | --- | --- |
| `https://traefik.playcluster.plone.org/` | The cluster's Traefik dashboard, on `play1` | `admin`, password from `vault.traefik.ui_basic_auth` |
| `https://portainer.playcluster.plone.org/` | Portainer, a web UI for the swarm, on `play1` | the administrator created with Portainer's setup token |
| `https://registry.playcluster.plone.org/v2/` | The registry API, on `play4` — what `docker` talks to | `ci` (push) or `deploy` (pull) |
| `https://registry.playcluster.plone.org:7443/` | The registry's web UI | `ci` or `deploy` |
| `https://registry.playcluster.plone.org:8443/dashboard/` | The dashboard of `play4`'s own Traefik — the trailing slash matters | `admin`, the same password as the cluster's Traefik |

The registry answers on three ports because zot serves its API and its web UI from one listener:
port 443 is kept for the API alone (`/v2/`), so `https://registry.playcluster.plone.org/` itself gives
a 404. Chapter 7 of the documentation explains the arrangement.

## Quick start

You need four servers running a clean Ubuntu 26.04, DNS names for them, and SSH access as `root`.
The [documentation](#documentation) explains each step; in short:

```shell
make install
```

Create a vault password and your own vault from the template (the committed `vault.yml` is
encrypted with the maintainer's password):

```shell
openssl rand -hex 32 > .vault_pass && chmod 600 .vault_pass
```

```shell
cp etc/vault.template.yml vault.plain.yml
```

Fill in `vault.plain.yml`, then encrypt it into place and remove the plain copy:

```shell
uv run ansible-vault encrypt --output inventory/group_vars/all/vault.yml vault.plain.yml && rm vault.plain.yml
```

Check the connection, then provision the manager first, the workers next, and the CI host last:

```shell
uv run ansible-playbook playbooks/_connect.yml
```

```shell
uv run ansible-playbook playbooks/setup.yml --limit cluster_manager
```

```shell
uv run ansible-playbook playbooks/setup.yml --limit cluster_workers
```

```shell
uv run ansible-playbook playbooks/setup_ci.yml --limit standalone
```

The documentation does the very first run one host at a time (`--limit play2.playcluster.plone.org`)
so that each run's output belongs to a single server; the groups above are the everyday form.

Before the first run, read chapter 1 on SSH keys, and chapter 5 for the GitLab runner token the
last command needs.

## Documentation

The documentation lives in `docs/`, as Markdown, and builds to HTML and PDF:

| Chapter | |
| --- | --- |
| 1 | SSH, keys and the two users |
| 2 | Ansible in one sitting — including your own vault and your own domain |
| 3 | Provision the manager |
| 4 | Provision the workers |
| 5 | The CI runner host |
| 6 | Connecting GitLab — the CI/CD variables |
| 7 | The registry in detail |

```shell
cd docs && make html
```

```shell
cd docs && make pdf
```

The Plone training includes these chapters as its reference section. They are maintained here
only; `make export-training` in `docs/` copies them into a checkout of the training.

## Repository layout

| Path | Holds |
| --- | --- |
| `inventory/hosts.yml` | The four hosts and their groups |
| `inventory/group_vars/` | All settings; `all/` for every host, `standalone/` for `play4` only |
| `playbooks/` | `setup.yml` for the swarm hosts, `setup_ci.yml` for `play4`, `deploy.yml` for stacks |
| `tasks/` | The steps the playbooks run, by area |
| `etc/stacks/` | Swarm stacks on the manager: Traefik, cronjob, Portainer |
| `etc/compose/` | The Docker Compose project on `play4`: zot and its Traefik |
| `etc/keys/` | SSH keys installed for the `plone` user — gitignored |
| `etc/vault.template.yml` | Placeholders for the vault |
| `docs/` | The documentation |

The swarm hosts get their services as swarm stacks, `play4` as a Compose project — chapter 7 explains
why, and how the two mirror each other.

This repository started from the Ansible setup that [cookieplone](https://github.com/plone/cookieplone)
generates for Plone projects.
