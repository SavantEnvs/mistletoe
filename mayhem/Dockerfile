# syntax=docker/dockerfile:1.7
#
# Commit-image Dockerfile (SMALL form) — builds ON TOP of the org base image
# (ghcr.io/mayhemheroes/base). The base provides debian + clang/llvm + libFuzzer + Python 3 + the
# identities + the build-contract ENV. mistletoe is a PURE-PYTHON Markdown parser with ZERO runtime
# dependencies, so this image installs the Atheris harness engine plus the test oracle's deps
# (pytest, parameterized, Pygments) into the base's system Python as root, then (as the non-root
# `mayhem` user) compiles the ELF launcher shims and runs the real unittest suite via
# build.sh / test.sh.
#
# Build (from the repo root):
#   DOCKER_BUILDKIT=1 docker build -f mayhem/Dockerfile \
#       --build-arg COMMIT=$(git rev-parse HEAD) \
#       [--build-arg SOURCE_DATE_EPOCH=$(date +%s)] \
#       -t mistletoe-commit:test .

FROM ghcr.io/savantenvs/base:20260621

ARG COMMIT
ARG REPO_URL
# SOURCE_DATE_EPOCH stays an ARG (available to build.sh); never bake it as ENV — an empty value
# breaks clang. build.sh unsets it when empty.
ARG SOURCE_DATE_EPOCH
# Declared for parity with the build contract; mistletoe has no native code to sanitize.
ARG SANITIZER_FLAGS

# Nothing may write into the /mayhem source tree as root: the non-root `mayhem` user has to be able
# to git-clean it afterwards (§6.2 item 16), and a stray root-owned __pycache__/*.pyc would fail that.
ENV PYTHONDONTWRITEBYTECODE=1

# mistletoe is imported from the CHECKOUT, not from site-packages. At this (2023) commit
# `mistletoe.__version__` is `1.0.2-SNAPSHOT`, and setup.py feeds it straight to setup(version=...);
# setuptools >= 66 validates versions against PEP 440 and aborts with
#   packaging.version.InvalidVersion: Invalid version: '1.0.2-SNAPSHOT'
# so `pip install .` cannot succeed here. Patching setup.py / __init__.py is not an option (a
# backport branch may not modify upstream files), and pinning an ancient setuptools would fight the
# base image. A pure-Python package with zero runtime deps does not need installing at all: putting
# /mayhem on PYTHONPATH makes `import mistletoe` resolve to the fuzzed source tree itself — which is
# exactly what a point-in-time reproduction wants, and is what the harness and the unittest suite
# both consume.
ENV PYTHONPATH=/mayhem

# Atheris (the libFuzzer harness engine — builds a native module against the base's clang/libFuzzer)
# plus the test oracle's deps (pytest + parameterized; Pygments for the contrib pygments renderer
# tests) go into the base's SYSTEM Python. PEP 668 marks that interpreter externally-managed, so
# --break-system-packages. This is the (networked) install step; build.sh deliberately does NOT
# pip-install, so its offline PATCH-tier re-run stays air-gapped.
USER root
RUN pip install --no-cache-dir --break-system-packages \
        atheris pytest parameterized 'Pygments>=2.11.2,<=2.16'

# Source enters from the build CONTEXT (the CI checkout): the FULL repo (incl. .git history)
# is copied to /mayhem as `mayhem`; build.sh builds in place.
WORKDIR /mayhem
COPY --chown=2000:2000 . /mayhem

# `import mistletoe` comes from $PYTHONPATH=/mayhem (see above) — no install step. Assert it here so
# a broken import fails the BUILD rather than surfacing as a Mayhem smoke-test failure hours later.
RUN python3 -c "import mistletoe, mistletoe.latex_renderer, mistletoe.core_tokens as c; \
    print('mistletoe', mistletoe.__version__, 'from', mistletoe.__file__); \
    assert mistletoe.__file__.startswith('/mayhem/'), mistletoe.__file__"

USER mayhem

# Compile the ELF launcher shims (Mayhem needs an ELF target, not a .py).
RUN bash mayhem/build.sh

# Run mistletoe's real unittest known-answer suite at build time; a failing suite fails the build.
RUN if [ -f mayhem/test.sh ]; then bash mayhem/test.sh; else echo "no mayhem/test.sh"; fi

LABEL org.opencontainers.image.source="${REPO_URL}" \
      org.opencontainers.image.revision="${COMMIT}" \
      rlenv.tier="commit" \
      rlenv.source_date_epoch="${SOURCE_DATE_EPOCH}" \
      rlenv.user="mayhem"

USER mayhem
WORKDIR /mayhem
