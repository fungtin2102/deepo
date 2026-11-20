# AGENTS.md

## Cursor Cloud specific instructions

Deepo is a **pure-Python (stdlib only)** Dockerfile generator. There are no third-party
dependencies, no `requirements.txt`/`setup.py`, no automated test suite, and no lint config.
"Running the app" means running the Python generator; "building" means regenerating the
`docker/Dockerfile.*` files and the GitHub Actions CI config.

### Key gotcha: `python` vs `python3`
The helper scripts (`scripts/build.sh`, the generated `scripts/gen-docker.sh`) hardcode the
`python` command, but the base image only ships `python3`. A `python -> python3` symlink at
`/usr/local/bin/python` is created during environment setup (persisted in the VM snapshot), so
the scripts work as written. If you ever hit `python: command not found`, recreate it with
`sudo ln -sf /usr/bin/python3 /usr/local/bin/python`.

### Generator import paths are cwd-relative
`generator/generate.py` does `from core.composer import Composer` and dynamically imports
`modules.<name>`, so it MUST be run from inside the `generator/` directory:
```bash
cd generator
python generate.py /tmp/Dockerfile.out pytorch jupyter python==3.6   # one framework set
python generate.py /tmp/Dockerfile.gpu tensorflow python==3.6 --cuda-ver 11.1 --cudnn-ver 8
```

### Regenerate everything (the "build")
```bash
cd scripts
bash build.sh   # runs make-gen-docker.py -> gen-docker.sh -> regenerates docker/* and make-ci.py -> CI yml
```
Use this as the correctness check: after `build.sh`, `git diff` on `docker/` should be empty
(generated Dockerfiles match what's committed). Note `scripts/make-ci.py` walks `os.listdir`,
so `.github/workflows/dockerimage.yml` is regenerated with non-deterministic job ordering; a
diff there that only reorders jobs (same job set, same line count) is expected and not a
regression — `git checkout` it to discard the no-op churn.

### Testing / lint
There is no unit-test or lint tooling in this repo. The real validation is: (1) the generator
produces a Dockerfile without error, and (2) `build.sh` reproduces the committed `docker/*`
files. Actually building the generated Docker images requires GPU/large downloads and is done
in GitHub Actions (`.github/workflows/dockerimage.yml`), not locally.
