# mdci.setup.feature-lancedb

LanceDB and a small embedding model, in a private environment of their own. OPTIONAL and general: any project may take it, and everything works without it. The first user is the Knowledge store of `myx.distro-agents`, one store per workspace beside each `MAGIC.md`.

Maintainer notes: `MAGIC.md`.

## Requires

- Nothing in `Requires:`.
- A workspace with `.local` (`MMDAPP`), and the local tools in it: the Python is theirs, `DistroLocalTools.fn.sh --install-embedded-python`. This feature asks that tool and never installs a Python itself.
- On FreeBSD amd64 the Python is the system's `python3.12` instead, with `python312 py312-pip py312-lancedb py312-tokenizers py312-numpy rust` from pkg. The install adds the missing ones when run as root, and otherwise refuses and names them.
- Internet for an online install (PyPI, Hugging Face), or a prepared folder for `--from-folder`.

## Where it installs, and where it does not

One path per platform: binary packages only, into the embedded Python's virtual environment. FreeBSD amd64 is the exception, as the table says.

| platform | result | package source |
|---|---|---|
| macOS arm64 | installs | `lancedb` wheel `cp310-abi3-macosx_11_0_arm64` from PyPI |
| Linux x86_64 (glibc, with AVX2 and FMA) | installs | `lancedb` wheel `manylinux_2_28_x86_64` from PyPI |
| Linux aarch64 (glibc) | installs | `lancedb` wheel `manylinux_2_28_aarch64` from PyPI |
| Intel macOS | does not install | no wheel exists |
| Linux with musl | does not install | no wheel exists |
| Linux x86_64 without AVX2 and FMA | does not install | the wheel dies with `Illegal instruction` |
| FreeBSD amd64 | installs | `py312-lancedb`, `py312-tokenizers` and `py312-numpy` from pkg, seen by the virtual environment (`--system-site-packages`); `model2vec` and `safetensors` by pip with `--no-deps`, `safetensors` built from source with pkg `rust` |
| FreeBSD aarch64 and other processors | does not install | the port `py-lancedb` is broken on aarch64 (lancedb issue 3764) |
| anything else | does not install | not covered |

Where it does not install, the install says why on standard error, writes nothing and exits 3; the check says `not supported here`. The Knowledge store then stays off there and reads `MAGIC.md` alone.

The embedding model is `potion-base-8M` of `minishlab` (MIT), run by `model2vec`: static, 256 dimensions, about 30 MB, no torch, CPU only. Pins: `lancedb 0.40.0`, `model2vec 0.10.0`, model revision `bf8b056651a2c21b8d2565580b8569da283cab23`.

## What it installs

All in `$LANCEDB_HOME`, which is `$MMDAPP/.local/lancedb` unless `LANCEDB_HOME` is set (absolute). Never the system Python, never a home folder, never a shell startup file. The one exception is FreeBSD amd64, where the install adds the missing pkg packages named under Requires to the system.

- `venv/` — the environment: `lancedb`, `model2vec` and what they need (numpy, pyarrow, tokenizers, safetensors, huggingface-hub and more), from binary packages alone, after both were resolved together. On FreeBSD amd64 it holds `model2vec` and `safetensors` only, and sees the rest from pkg.
- `model/potion-base-8M/` — the model files `model2vec` reads (not the ONNX copy).
- `wheelhouse/` — the packages as downloaded. With `model/potion-base-8M/` it is what `--from-folder` takes on a host with no internet.
- `licenses/` and `packages.txt` — the licence files of every package, and name, version and licence of each, read from the wheels. The list is printed before anything is placed.
- `hf/`, `cache/`, `tmp/` — the caches and temp folder of the processes it runs, so nothing lands in `~/.cache`.
- `.mdci-lancedb-home` — marks the folder as this feature's. `lancedb.env` — written last; it is there only when the install is whole.

A folder that exists, is not empty and has no `.mdci-lancedb-home` is refused and left as it is.

## Commands

Fragments in `host/install/`, run with `sh`, the environment of a workspace (`MMDAPP`) or `LANCEDB_HOME` set:

- `feature-lancedb.install.txt [--from-folder <folder>]` — installs. It prints the environment's Python on standard output, one line, and everything else on standard error. Safe to run again: what stands at the pinned versions is left as it is. A failed run leaves nothing of its own. Exit 0 installed, 1 refused or failed, 3 platform not covered.
  - `--from-folder <folder>`: no download. The folder holds the `.whl` files and the `potion-base-8M/` folder, copied from an online run's `wheelhouse/` and `model/` made on the same system and processor.
- `feature-lancedb.uninstall.txt` — removes `$LANCEDB_HOME`. It does not touch the embedded Python (other tools use it) and does not touch any store (`MAGIC.LanceDB/` holds the knowledge of its workspace). Exit 0, or 1 if the folder is not this feature's.
- `feature-lancedb.check.txt` — reports on standard output:
  - `lancedb: installed | not installed | broken | not supported here`
  - `platform`, `home`, `size` in MB, `versions`, `model`
  - `query: ok | failed` — a scratch table of three sentences is made in `$LANCEDB_HOME/tmp`, embedded with the model, and the nearest sentence to a question must be the right one; the scratch folder is removed.
  - Exit 0 installed and the query works, 1 not installed, 2 broken, 3 not covered.
- `feature-lancedb.common.txt` — shared by the three; not run by itself.

There is no `image-install:` hook in `project.inf`: this is a workspace feature, not a host-image step. A host profile that wants it on a server runs the install fragment with `LANCEDB_HOME` set.

## The contract the Knowledge store calls

A workspace finds the environment and the model by path, never by `PATH`, and never by asking a package manager:

1. **Home**: `$LANCEDB_HOME`, else `$MMDAPP/.local/lancedb`.
2. **Installed** means `$LANCEDB_HOME/lancedb.env` exists. Absent: the feature is off, and the caller carries on without it. `lancedb.env` is plain `KEY=value` and can be sourced: `LANCEDB_VERSION`, `MODEL2VEC_VERSION`, `LANCEDB_MODEL_NAME`, `LANCEDB_MODEL_REVISION`, `LANCEDB_MODEL_DIM` (256), `LANCEDB_PLATFORM`.
3. **Python**: `$LANCEDB_HOME/venv/bin/python`, run isolated (`-I`) with `HF_HUB_OFFLINE=1`, `HF_HOME=$LANCEDB_HOME/hf` and `TMPDIR=$LANCEDB_HOME/tmp`.
4. **Model**: the folder `$LANCEDB_HOME/model/potion-base-8M`, loaded by path with the net off: `StaticModel.from_pretrained("<that folder>")`, then `.encode([...])`, vectors of 256 floats. The name is never a Hub id at run time.
5. **Is it working**: `sh feature-lancedb.check.txt`; the exit code is the answer, the lines are the detail.
6. **Install or remove**: the install and uninstall fragments above, by full path under `$MDLT_ORIGIN/myx/mdci-packages-myx/setup.feature-lancedb/host/install/`.
7. **The stores are the caller's**: a store is a directory (`MAGIC.LanceDB/`) beside its `MAGIC.md`, and is never inside `$LANCEDB_HOME`. Practice for stores kept in git: one writer at a time; compact before a commit (`table.optimize(cleanup_older_than=timedelta(0))`); never merge two writers' stores, take one side whole and ingest again.

## Licences

Apache-2.0 for `lancedb` and `pyarrow`; MIT for `model2vec` and the model; `numpy` is BSD-3-Clause and others, `tqdm` is MIT and MPL-2.0, `typing-extensions` is PSF. The authority is the list the install prints and keeps in `packages.txt`, read from each wheel.
