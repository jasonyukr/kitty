# Building and installing kitty on this machine

Personal build notes for `ol9` (Oracle Linux 9). Verified against kitty **0.48.0**.

This documents a **self-contained install** at `/opt/kitty`: kitty runs against the
vendored dependency tree that kitty's own build system downloads, not against
distro libraries. Nothing needs to be installed on the host.

Upstream's `docs/build.rst` covers the plain `./dev.sh build` path. It does **not**
cover producing an installable, relocatable `linux-package` against vendored deps —
that is what the rest of this file is for.

---

## 0. Why do we need to build ourself?

The official install method fails on ol9 host machine due to GLIBC version mismatch (since v0.48.0).
```
jinhyu@ol9:~% curl -L https://sw.kovidgoyal.net/kitty/installer.sh | sh /dev/stdin
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4571  100  4571    0     0  13210      0 --:--:-- --:--:-- --:--:-- 13210
Downloading from: https://github.com/kovidgoyal/kitty/releases/download/v0.48.0/kitty-0.48.0-x86_64.txz

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 30.5M  100 30.5M    0     0  15.9M      0  0:00:01  0:00:01 --:--:-- 25.2M
Installing to /home/jinhyu/.local/kitty.app
/home/jinhyu/.local/kitty.app/bin/kitty: /lib64/libm.so.6: version `GLIBC_2.35' not found (required by /home/jinhyu/.local/kitty.app/bin/../lib/libpython3.14.so.1.0)
jinhyu@ol9:~%
```

---

## 1. Why the build runs in a container

The host is missing the X11/DBUS development headers kitty links against:

| pkg-config | RPM |
|---|---|
| `xrandr` | `libXrandr-devel` |
| `xinerama` | `libXinerama-devel` |
| `xcursor` | `libXcursor-devel` |
| `xkbcommon-x11` | `libxkbcommon-x11-devel` |
| `dbus-1` | `dbus-devel` |

Rather than installing these on the host, build inside the dev container, which
already has them (see the *"Install tools for kitty build"* block in
`~/container/podman/oraclelinux-9_dev/Containerfile`), plus Go in `/usr/local/go`.

Everything else kitty needs — Python 3.14, harfbuzz, freetype, lcms2, libpng,
libxxhash, openssl — comes from `dependencies/`, downloaded by kitty's build
system. Those are *not* expected from the distro, which is why `pkg-config`
reporting them as missing on the host is harmless.

`~/github` is bind-mounted into the container, so build output lands in the real
repo and the resulting binaries run on the host.

Enter the container:

```sh
~/container/podman/scripts/oraclelinux9.sh
```

The shell prompt shows `oraclelinux9` inside the container and `ol9` on the host —
that is the quickest way to tell where you are.

---

## 2. Dev build (run from source, no install)

In the container:

```sh
cd ~/github/jasonyukr/kitty
export PATH=/usr/local/go/bin:$PATH     # needs Go >= the version in go.mod
./dev.sh build
```

Produces `kitty/launcher/kitty` and `kitty/launcher/kitten`, runnable directly,
including from the host. Good enough for testing a change; not what gets installed.

To refresh the vendored deps (do this occasionally, per upstream):

```sh
./dev.sh deps
```

---

## 3. Package build (what actually gets installed)

### 3.1 One-time: documentation dependencies

`setup.py linux-package` builds man pages and HTML docs via sphinx, and **aborts
the entire package if sphinx is missing** — it is not a warning. Sphinx is only
skipped when `docs/_build/html` already exists or `--skip-building-kitten` is
passed (`setup.py:1511-1526`).

```sh
./dev.sh deps -for-docs
```

Installs sphinx and friends into `dependencies/linux-amd64/bin`. Re-run after any
`./dev.sh deps`, which wipes them.

### 3.2 Build the package

```sh
cd ~/github/jasonyukr/kitty
rm -f linux-package/bin/kitty          # see "stale launcher" below — do not skip
root=$PWD/dependencies/linux-amd64
env -u PYTHONPATH \
  PATH=$root/bin:/usr/local/go/bin:$PATH \
  PYTHONHOME=$root \
  LD_LIBRARY_PATH=$root/lib \
  PKG_CONFIG_PATH=$root/lib/pkgconfig \
  DEVELOP_ROOT=$root \
  CPPFLAGS='-DSET_PYTHON_HOME=\"../../../dependencies/linux-amd64\"' \
  LDFLAGS='-Wl,--disable-new-dtags -Wl,-rpath,$ORIGIN/../../dependencies/linux-amd64/lib' \
  $root/bin/python setup.py linux-package
```

Note the quoting: `LDFLAGS` is **single-quoted** so `$ORIGIN` reaches the linker
literally, and the `\"` in `CPPFLAGS` are required so the macro arrives as a C
string literal.

### 3.3 Verify before installing

```sh
objdump -x linux-package/bin/kitty | grep -i rpath
#   RPATH   $ORIGIN/../../dependencies/linux-amd64/lib

strings linux-package/bin/kitty | grep '\.\./\.\./\.\./dependencies'
#   ../../../dependencies/linux-amd64

./linux-package/bin/kitty +runpy 'print("python ok")'
```

Use `+runpy`, **not** `--version`. `--version` is answered by the C launcher before
Python is initialized, so it succeeds even when the install is fatally broken.
`+runpy` is the check that actually exercises Python startup.

---

## 4. Install to /opt/kitty

Run on the **host** — `/opt/kitty` is not mounted in the container. Replace
`<OLD>` with the version currently installed (`/opt/kitty/linux-package/bin/kitty --version`):

```sh
cd ~/github/jasonyukr/kitty
sudo mv /opt/kitty/dependencies   /opt/kitty/___v<OLD>_dependencies
sudo mv /opt/kitty/linux-package  /opt/kitty/___v<OLD>_linux-package
sudo cp -a dependencies           /opt/kitty/dependencies
sudo cp -a linux-package          /opt/kitty/linux-package
/opt/kitty/linux-package/bin/kitty +runpy 'print("python ok")'
```

`mv` rather than `rm` — the old tree stays intact as a rollback. Backup dirs are
named for the version they *contain*.

`PATH` already leads with `/opt/kitty/linux-package/bin`, so just restart kitty.

Roll back by swapping both pairs back. Delete old `___v*` backups once the new
version has proven itself.

**`dependencies/` and `linux-package/` must always be installed as a pair** — see
below.

---

## 5. Why the two injected flags are needed

`setup.py` supports two relevant bundle types, and neither does what is wanted here:

- **`develop`** — sets the RPATH and `SET_PYTHON_HOME` for the vendored deps, but
  hardcodes an absolute path to *this repo* as the kitty lib dir, so it is not
  relocatable to `/opt`.
- **`linux-package`** — relocatable (bakes `../lib/kitty`), but assumes harfbuzz,
  libxxhash, Python etc. are installed system-wide. On this host such a build dies
  immediately with `libxxhash.so.0: cannot open shared object file`.

The install needs *both* halves: `linux-package`'s relocatable layout plus
`develop`'s pointers into the vendored tree. `setup.py` has no bundle type for
that, so the two missing pieces are injected through env vars it already honours
(`setup.py:1378` for `LDFLAGS`, `setup.py:1418` for `CPPFLAGS`):

**`LDFLAGS` → RPATH `$ORIGIN/../../dependencies/linux-amd64/lib`**
Lets the launcher find `libpython3.14.so.1.0` and the other vendored shared
libraries. From `/opt/kitty/linux-package/bin/` this resolves to
`/opt/kitty/dependencies/linux-amd64/lib`. `--disable-new-dtags` forces a real
`RPATH` rather than `RUNPATH`, because `RUNPATH` is not searched for transitive
dependencies.

**`CPPFLAGS` → `SET_PYTHON_HOME="../../../dependencies/linux-amd64"`**
Tells the launcher where the Python stdlib lives. `main.c:169` builds the value as
`lib_dir + "/" + SET_PYTHON_HOME`; from `/opt/kitty/linux-package/lib/kitty` that
resolves to `/opt/kitty/dependencies/linux-amd64`. Without it the launcher starts,
finds no stdlib, and dies with:

```
Fatal Python error: Failed to import encodings module
ModuleNotFoundError: No module named 'encodings'
```

Both paths are **relative**, which is what makes the tree relocatable — and also
why the two directories must stay siblings inside `/opt/kitty`. Copying a new
`linux-package` next to an old `dependencies` breaks both.

---

## 6. Gotchas

**Stale launcher.** `setup.py` skips relinking when `linux-package/bin/kitty` is
newer than its sources. Changing only env vars does not invalidate it, so
`CPPFLAGS`/`LDFLAGS` edits are silently ignored and you get a binary missing the
RPATH or `SET_PYTHON_HOME` with no error at all. Always `rm linux-package/bin/kitty`
first. Symptom: `objdump`/`strings` show nothing, and the timestamp predates the run.

**`sphinx-build: command not found`.** `$root/bin` missing from `PATH`. The docs
Makefile invokes `sphinx-build` bare, so `PYTHONHOME` alone is not enough.

**`The go tool was not found`.** `/usr/local/go/bin` missing from `PATH` — easy to
drop when overriding `PATH` for sphinx. Only affects the `kitten` rebuild; the rest
of the package still completes.

**ccache `Permission denied` in a custom podman invocation.** Mounting
`~/.cache/go-build` makes podman create `~/.cache` owned by root inside the
container. Mount `~/.cache/ccache` too. `run.sh` already does; ad-hoc `podman run`
lines may not.

**Host `pkg-config` reports fontconfig/freetype/harfbuzz/lcms2/libpng/libxxhash as
missing.** Expected and harmless — they come from `dependencies/`, not the distro.

**Matplotlib config-dir warnings during the docs build.** Cosmetic; the docs build
fine.

---

## 7. Quick reference

```sh
# container
cd ~/github/jasonyukr/kitty
./dev.sh deps -for-docs                      # once, after any ./dev.sh deps
rm -f linux-package/bin/kitty
root=$PWD/dependencies/linux-amd64
env -u PYTHONPATH PATH=$root/bin:/usr/local/go/bin:$PATH PYTHONHOME=$root \
  LD_LIBRARY_PATH=$root/lib PKG_CONFIG_PATH=$root/lib/pkgconfig DEVELOP_ROOT=$root \
  CPPFLAGS='-DSET_PYTHON_HOME=\"../../../dependencies/linux-amd64\"' \
  LDFLAGS='-Wl,--disable-new-dtags -Wl,-rpath,$ORIGIN/../../dependencies/linux-amd64/lib' \
  $root/bin/python setup.py linux-package
./linux-package/bin/kitty +runpy 'print("python ok")'

# host
sudo mv /opt/kitty/dependencies  /opt/kitty/___v<OLD>_dependencies
sudo mv /opt/kitty/linux-package /opt/kitty/___v<OLD>_linux-package
sudo cp -a dependencies  /opt/kitty/dependencies
sudo cp -a linux-package /opt/kitty/linux-package
/opt/kitty/linux-package/bin/kitty +runpy 'print("python ok")'
```
