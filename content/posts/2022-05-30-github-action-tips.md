---
title: "Github Action Tips"
date: 2022-05-30T10:00:25+02:00
draft: true
categories:
  - github
  - github-actions
---

# how to debug events

```yaml
- name: debig GitHub context
  env:
    GITHUB_CONTEXT: ${{ toJson(github) }}
  run: |
    echo "$GITHUB_CONTEXT"
```

```yaml
- name: debig GitHub context
  run: cat /home/runner/work/_temp/_github_workflow/event.json
```

# how to run build steps with own custom container

Sometimes you have to run integration tests with own container
which contains all devel-dependecies you need.
Something crazy like this:
```yaml
name: Pull Request Workflow
on:
  ...

env:
  ...

jobs:
  own-container-jobs:
    runs-on: ubuntu-latest
    container: docker.io/yourpublicaccount/builder-container:1.0.0
    services:
      postgres:
        ...
      redis:
        ...

    steps:
      - name: Check out repository code
        uses: actions/checkout@v2

      - uses: actions/cache@v2
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('*/requirements/path.txt') }}

      - name: Build seed VENV
        run: |
          # export PATH needs only if we have to build Psycopg2
          # Psycopg2 uses pg_config to find the libraries at build time
          export PATH=/usr/pgsql-10/bin:$PATH
          cd seed_dir
          python3.6 -m venv venv
          ./venv/bin/pip install -r requirements/path.txt

      - name: Seed DB
        run: |
          cd seed_dir
          ./venv/bin/python ...
          ./venv/bin/alembic ...

      - name: Build work VENV
        run: |
          cd work_dir
          python3.8 -m venv venv
          ./venv/bin/pip install -r requirements/path.txt

      - name: Run tests
        run: |
          cd work_dir
          ./venv/bin/pytest -v
```
If your contaner runs as a non-root user, you need adds some github-related things.
See comments bellow.
```dockerfile
ARG CODE_VERSION
FROM docker.io/redhat/ubi8:latest
ENV CODE_VERSION=${CODE_VERSION}
RUN dnf install -y --setopt=install_weak_deps=False \
        bzip2 bzip2-devel gcc-c++ git glibc-langpack-en \
        libffi-devel libpq-devel libxml2-devel libxslt-devel \
        python36-devel python38 python38-devel \
        ... \
        zlib-devel && \
        dnf clean all && rm -rf /var/cache/dnf

# RUN useradd -rm -d /github/home -s /bin/bash -g root -G sudo -u 1001 builder
# We need non-root user with uid 1001
# Because github overrides HOME environ, and creates /github/home dir with uid 1001
# - https://github.com/actions/runner/issues/863
# - https://github.com/actions/runner/issues/1525
RUN mkdir /github && \
    groupadd --system --gid 1001 builder && \
    useradd --system --create-home --home-dir /github/home --shell /bin/bash --uid 1001 --gid 1001 builder
USER builder
WORKDIR /github/home
```
