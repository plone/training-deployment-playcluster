---
myst:
  html_meta:
    "description": "Documentation for the playcluster Ansible setup."
    "property=og:description": "Documentation for the playcluster Ansible setup."
    "property=og:title": "playcluster"
    "keywords": "playcluster, documentation, Ansible, Docker Swarm"
---

# playcluster

Welcome to the documentation for playcluster.

This repository provisions and operates a Plone cluster: Ansible playbooks that
turn bare Linux servers into a Docker Swarm running Plone, with a GitLab CI
runner and a container registry alongside.

The documentation walks through the whole setup from scratch, in six
chapters.

% The chapters are listed here rather than in the overview's own index, so each
% becomes a top-level chapter of the PDF. Nested inside that page they would
% attach to whichever section holds the toctree, and the overview's own
% sections would sort after them.

```{toctree}
:caption: Building the cluster
:maxdepth: 2
:hidden: true

tutorials/cluster-setup/index
tutorials/cluster-setup/1-ssh-and-users
tutorials/cluster-setup/2-ansible
tutorials/cluster-setup/3-manager
tutorials/cluster-setup/4-workers
tutorials/cluster-setup/5-ci-runner
tutorials/cluster-setup/6-gitlab
```

```{toctree}
:caption: Appendices
:maxdepth: 2
:hidden: true

glossary
```
