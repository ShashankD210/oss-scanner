# Threat model

## What this project does and where untrusted input enters

This repo holds the Dockerfiles for Lium pod images (`templates/<name>/`) and the files they share (`scripts/`).
Lium rents GPU pods to paying customers ("renters"). A pod is a container on a host that a third-party provider
("miner") owns. Each image starts with `scripts/start.sh` (or a template `entrypoint.sh` that calls it). That script
starts nginx, runs `/pre_start.sh`, starts sshd, starts Jupyter Lab when `JUPYTER_PASSWORD` is set, and runs
`/post_start.sh`. Web-UI images (vscode, comfyui, stable-diffusion, oobabooga, ...) ship `scripts/proxy/nginx.conf`,
which proxies public ports to apps on localhost.

Untrusted input:

- **Network traffic to exposed ports.** SSH (22), Jupyter (8888), the nginx proxy ports (3001, 7861, 8001, 8081,
  9091, 7270) and the apps behind them, and the metrics sidecars in `dolphin`, `engy` and `pearl-miner`. Anyone on
  the Internet can reach these ports.
- **Environment variables set at pod creation.** Examples: `JUPYTER_PASSWORD`, `PUBLIC_KEY`, `METRICS_TOKEN`,
  `LIUM_WIREGUARD_CONF_B64`, `LIUM_CLUSTER_SSH_KEY_B64`. The renter or the platform sets them. Treat values as
  attacker-controlled when they reach a shell, a config file or a command line.
- **Other tenants.** In a group rental (`templates/lium-cluster`) several pods share a WireGuard overlay and one
  SSH key. Pods of other renters on the same host or network are untrusted.

Trusted: the renter inside their own pod (they have root there by design), the Lium validator and backend, and
the Docker build itself.

## Components that matter most / least

- Most: `scripts/start.sh`, `scripts/proxy/nginx.conf`, the entrypoints and SSH setup in `templates/*/entrypoint.sh`
  and `templates/pytorch/pytorch-entrypoint.sh`, `templates/lium-cluster/` (overlay, shared SSH key,
  `lium-rdma-runc.py` device allowlist), and the token checks in the `metrics_sidecar.py` files.
- Medium: `pre_start.sh` / `post_start.sh` files, Dockerfiles (what ends up in the image, file modes, sudoers).
- Least: `scripts/build-smoke.sh`, `.github/`, READMEs and notebooks. They do not ship in a pod.

## How to exercise it

- The image does not build the real pod images (they are many GB). It lays out the pod files instead:
  `/start.sh`, `/etc/nginx/nginx.conf`, `/usr/share/nginx/html/readme.html`, `/workspace`, and installs openssh-server,
  nginx and Jupyter Lab. Run `JUPYTER_PASSWORD=x /start.sh &` or `REQUIRE_NGINIX=true /start.sh &`, then use
  `curl`, `ssh` and `nc` against localhost.
- `nginx -t` checks the proxy config. `shellcheck` and `hadolint` are installed.
- Repo tests: `templates/pytorch/tests/test-entrypoint.sh`; `python3 <file>` for `templates/dolphin/tests/test_*.py`,
  `templates/engy/tests/test_*.py` and `templates/pearl-miner/tests/test_sidecar.py`; `python3 -m pytest
  test_lium_fabric_env.py test_lium_rdma_runc.py` in `templates/lium-cluster`. Tests that need Docker (`test_peer_login.sh`,
  `test-install-chain.sh`) do not run here.

## How you rate severity

- **Critical:** unauthenticated access to a pod's shell, SSH or web UI (auth bypass, a missing or default password
  or token); one tenant reaching another tenant's pod or data; a secret (key, token, password) baked into a
  published image layer; a pod reaching control of the host.
- **High:** a renter's secret (env var, token) leaked to logs, other pods or the network; command injection from an
  env var that the platform, not the renter, controls; an SSH key accepted from outside its allowed source.
- **Medium:** nginx config flaws that expose a local-only service, weak file modes on keys, CSRF or open origins
  on a web UI.
- **Low:** denial of service of one's own pod, crashes in start scripts.

## Anything to leave alone

- CVEs in upstream base images (CUDA, Ubuntu, PyTorch) and in third-party apps (ComfyUI, code-server, Jupyter,
  Ollama, ...). Report only issues in this repo's own files and in how we configure those apps.
- Root inside the pod. The renter owns the pod and has root by design.
- Things the renter does to their own pod with their own env vars or commands.
