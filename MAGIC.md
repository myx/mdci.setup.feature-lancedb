# MAGIC.md — setup.feature-lancedb

Maintainer notes. Users read `README.md`. How the packages compose: `../MAGIC.md`.

## Not yet run anywhere

- Written on 2026-10-26 without a run: only `sh -n` and `bash -n` were done. Nothing was installed, downloaded or executed, so these are unmeasured. They are the pip resolution of `lancedb==0.40.0` with `model2vec==0.10.0` together, and `huggingface_hub.snapshot_download` with `local_dir` and `allow_patterns` at `huggingface-hub>=1.0`. They also cover `StaticModel.from_pretrained` on a local folder, and the `table.search(vector)` call of the test query. The first live run on macOS arm64 settles them.

## Different from its siblings on purpose

- The other `setup.feature-*` packages are `image-install:` fragments for a FreeBSD or Ubuntu host image, run by the generator under `( set -e ; eval … )`. This one has no `image-install:` line: it installs into a workspace's `.local`, not into a host, and it runs on macOS laptops. Its three fragments (`install`, `uninstall`, `check`) are scripts run with `sh`, and do not use `set -e`; each step is checked and undone by hand.
- Nobody had written `uninstall` or `check` fragments before. The names follow the sibling `host/install/<name>.<step>.txt` shape.
- If a requiring host profile later wants it on a server, hooking `feature-lancedb.install.txt` as `image-install:exec-update-before:` works only after the fragment is given `LANCEDB_HOME` there. A host image has no `MMDAPP`.

## The Python is not ours

- The install asks `DistroLocalTools.fn.sh --install-embedded-python` by full path and uses the one line it prints. The path comes from `$MDLT_ORIGIN/myx/myx.distro-.local/sh-scripts/`. That script runs as a program only from a path ending `/myx/myx.distro-.local/sh-scripts/DistroLocalTools.fn.sh`. Where that refuses, this refuses. Do not add a second way to get a Python.
- The one exception is FreeBSD amd64: the base is pkg's `python3.12` (the ports' `PYTHON_DEFAULT`), and the venv is made `--system-site-packages --without-pip`. The port builds Python `--without-ensurepip`, so pip comes from `py312-pip`.
- The embedded Python is shared with other tools, so uninstall never removes it.

## Pins and where they come from

- `lancedb 0.40.0`: the pin of `AgentsTools.InstallKnowledgeStore.include` in `myx.distro-agents` and of the research (`20261021-refactorings.inputs/research.md`, item 1). Raise it only with the store's `AgentsKnowledgeStore.py`, whose full-scan call is written against it.
- `model2vec 0.10.0`: a pure-Python wheel, read from PyPI on 2026-10-26. Its requirements: jinja2, joblib, numpy, safetensors, tokenizers, tqdm, `huggingface-hub>=1.0.0`.
- Model revision `bf8b056651a2c21b8d2565580b8569da283cab23`: the revision of `minishlab/potion-base-8M` the Hub API reported on 2026-10-26 (MIT, 7,559,168 float32 parameters, so about 30 MB). Only the files `model2vec` reads are fetched; the ONNX copy is left out.
- No `--require-hashes`. The research recommends a hashed requirements file; no hash was computed (that needs a download). Until then the packages are version-pinned, not hash-pinned, and the licence list is read from whatever wheels came.

## Platforms

- Wheels exist on PyPI for macOS arm64 and Linux glibc (manylinux_2_28) on x86_64 and aarch64, and not for Intel macOS, musl or FreeBSD. x86_64 needs AVX2 and FMA. `LancedbPlatform` tests exactly that and nothing more, and the post-install probe catches what it cannot see.
- The AVX2 test reads `/proc/cpuinfo`; where that is unreadable it lets the probe decide.
- FreeBSD amd64: `LANCEDB_FREEBSD_PKGS` come from pkg. They are `py312-lancedb` 0.40.0, `rust`, and every runtime `Requires-Dist` of model2vec 0.10.0 (PyPI, read 2026-10-26). Each comes from its port: numpy, tokenizers>=0.20 (port 0.23.3), huggingface-hub>=1.0.0 (port 2.3.0), tqdm, Jinja2, joblib. Missing ones are installed as root, or the install refuses and names the `pkg install` line. pip adds `model2vec` and `safetensors` with `--no-deps`. `safetensors` is built from source: its port pulls in pytorch. `CARGO_HOME` stays inside the home folder.
- FreeBSD aarch64 stays off: the port is broken there (upstream issue 3764).
- `safetensors` is pinned to 0.8.0 (PyPI, 2026-06-09, sdist, no runtime dependencies, Python >=3.10); model2vec accepts any version.
- Not yet run on FreeBSD: the probe after the install is the first real proof.

## The model never leaves the folder

- `HF_HOME`, `XDG_CACHE_HOME` and `TMPDIR` point inside the home folder in every run, so nothing is written to `~/.cache`. Everything but the one download runs with `HF_HUB_OFFLINE=1`.
- `snapshot_download(local_dir=…)` may leave a `.cache` folder inside the model folder; that is Hub metadata and harmless.

## The home folder is marked

- `.mdci-lancedb-home` is what lets uninstall use `rm -rf` safely on a path that `LANCEDB_HOME` may have set to anything. Do not remove the marker check.
- `lancedb.env` is written last and removed first, so its presence is the only test of a whole install. The check fragment treats a folder without it as not installed, whatever else is in it.
