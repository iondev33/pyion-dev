# Portable pyion regression tests (Linux / WSL2 / macOS)

A generalized, host-independent way to run pyion's two-node protocol tests in
Docker. It reuses the existing per-protocol test assets under
`tests/<proto>_tests/` (node configs, `start`, `tx.py`, `rx.py`) and works from
any checkout location — no hardcoded host paths.

> This does **not** replace `tests/<proto>_tests/test_network.yaml` (Marc
> Sanchez's original Windows/Docker-Desktop setup); those are left untouched so
> that workflow keeps working. This is a parallel, portable entry point built
> on the same test assets.

## Prerequisites

- Docker Engine with the Compose v2 plugin (`docker compose version`).
- On WSL2: Docker Desktop with WSL integration, or Docker installed inside the
  WSL distro. Nothing else — ION and pyion are built inside the image/container.

## What it does

- `docker/Dockerfile` builds an image with **ION-DTN 4.2.0-b** (cloned from the
  public repo), the C toolchain, and Python.
- pyion itself is built **at container start** from your mounted working copy,
  so you can edit pyion and re-run without rebuilding the image.
- `docker/compose.yaml` brings up two nodes — `pyion_node1` (196.128.0.101) and
  `pyion_node2` (196.128.0.102) — on an isolated bridge network, matching the
  IPs already baked into the node configs.

## Run

Pick a protocol: `bp`, `cfdp`, `ltp`, or `mem`.

```bash
# from the repo root
docker/run.sh bp
# equivalently:
PYION_TEST=bp docker compose -f docker/compose.yaml up --build
```

The first run builds ION from source (~10–15 min); later runs are cached.

Once both nodes print `... up`, drive the test from two more shells
(**receiver first**):

```bash
docker exec -it pyion_node2 bash -lc 'cd /work/bp_tests && python3 rx.py'
docker exec -it pyion_node1 bash -lc 'cd /work/bp_tests && python3 tx.py'
```

`rx.py` prints received data and throughput; `tx.py` sends the test traffic.

Each node runs from a **container-internal copy** of the test dir
(`/work/<proto>_tests`), not the mounted repo — so ION's runtime log writes
and the scripts' `chmod +x` never dirty your working tree. pyion itself is
still built from your mounted working copy. Build artifacts it leaves in the
repo (`build/`, `*.so`, `pyion.egg-info/`) are already git-ignored.

Tear down:

```bash
docker/run.sh bp down
# or: docker compose -f docker/compose.yaml down
```

## Customizing

- **ION version** — override the build arg:
  ```bash
  docker compose -f docker/compose.yaml build \
    --build-arg ION_REF=ion-open-source-4.2.0-a.2
  ```
- **BP version** — set `PYION_BP_VERSION` (defaults to `BPv7`) in
  `docker/compose.yaml` or the environment.

## Notes

- The 196.128.0.0/16 subnet and per-node IPs come from the existing node
  configs (`node.bprc` UDP inducts/outducts, `node.ltprc` spans); they are kept
  as-is so the shared configs work unchanged.
- Containers run `privileged` because ION uses SysV shared memory / semaphores.
- `python3 setup.py ... install` is used to match the existing test flow. If a
  future base image rejects it (PEP 668), switch the compose command to
  `pip install --break-system-packages .`.
