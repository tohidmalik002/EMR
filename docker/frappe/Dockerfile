# Runtime for the demo-bench Frappe bench.
#
# This image deliberately contains NO application code. The bench's nine apps are
# bind-mounted from ./demo-bench/apps at run time, because they cannot be cloned:
# none of them has an `origin` remote, several sit on local feature branches, and
# `emr_sso` has no remote at all plus uncommitted work. The only copy that exists
# is the one on disk.
#
# Python 3.14 is not a preference — apps/frappe/pyproject.toml pins
# requires-python = ">=3.14,<3.15".
FROM python:3.14-bookworm

# Frappe v16 requires Node >=24 (apps/frappe/package.json "engines"). On 22,
# `bench watch` exits immediately with an incompatible-engine error and bench
# start then tears down every other process with it. This is the same reason
# the host Procfile prefixes PATH with nvm's v24.
ARG NODE_MAJOR=24

# Frappe's build/runtime dependencies:
#   libmariadb-dev + default-mysql-client -> mysqlclient, `bench --site ... mariadb`
#   libffi/libssl                         -> cryptography
#   libxml2/libxslt                       -> lxml
#   libjpeg/zlib/libwebp/freetype         -> Pillow
#   redis-tools                           -> bench's redis health checks
#   git                                   -> frappe pins gunicorn from a git URL
RUN apt-get update && apt-get install -y --no-install-recommends \
      git curl ca-certificates gnupg xz-utils \
      build-essential pkg-config \
      libmariadb-dev default-mysql-client \
      libffi-dev libssl-dev \
      libxml2-dev libxslt1-dev \
      libjpeg-dev zlib1g-dev libwebp-dev libfreetype6-dev \
      redis-tools \
      fonts-liberation fonts-dejavu-core \
      tini \
    && rm -rf /var/lib/apt/lists/*

# Node is needed for `bench socketio` (the realtime server) and `bench watch`,
# which also builds frontend assets.
RUN curl -fsSL "https://deb.nodesource.com/setup_${NODE_MAJOR}.x" | bash - \
    && apt-get install -y --no-install-recommends nodejs \
    && rm -rf /var/lib/apt/lists/* \
    && npm install -g yarn

# Frappe needs the Qt-patched wkhtmltopdf; the plain Debian package renders
# headers/footers and page breaks incorrectly. Fall back to the distro build
# rather than failing the image, and say which one landed.
RUN set -eux; \
    arch="$(dpkg --print-architecture)"; \
    url="https://github.com/wkhtmltopdf/packaging/releases/download/0.12.6.1-3/wkhtmltox_0.12.6.1-3.bookworm_${arch}.deb"; \
    if curl -fsSL -o /tmp/wkhtmltox.deb "$url"; then \
      apt-get update && apt-get install -y --no-install-recommends /tmp/wkhtmltox.deb \
      && rm -rf /var/lib/apt/lists/* /tmp/wkhtmltox.deb; \
      echo "wkhtmltopdf: patched-qt build installed"; \
    else \
      apt-get update && apt-get install -y --no-install-recommends wkhtmltopdf \
      && rm -rf /var/lib/apt/lists/*; \
      echo "wkhtmltopdf: WARNING - fell back to the unpatched Debian package"; \
    fi

# Frappe v16 renders print previews and PDFs with a headless Chromium it downloads into
# the bench. Without these shared libraries it exits at once and every preview fails with
# "Chromium took too long to start".
RUN apt-get update && apt-get install -y --no-install-recommends \
      libnss3 libnspr4 libdbus-1-3 libatk1.0-0 libatk-bridge2.0-0 libatspi2.0-0 \
      libxcomposite1 libxdamage1 libxfixes3 libxrandr2 libgbm1 libxkbcommon0 libasound2 \
    && rm -rf /var/lib/apt/lists/*

# The bench CLI lives outside the bench's own virtualenv, as on a normal install.
RUN pip install --no-cache-dir frappe-bench

# Run as a real user whose uid matches the host's, not as root.
# The bench's apps/ and sites/ are bind-mounted from the host, and bench writes
# into them (logs, *.egg-info, sites/assets). As root those files come back
# root-owned in your source tree and you can no longer edit them.
ARG UID=1000
ARG GID=1000
RUN groupadd -g "$GID" frappe 2>/dev/null || true; \
    useradd -m -u "$UID" -g "$GID" -s /bin/bash frappe 2>/dev/null || true

# Creating these in the image, owned by frappe, is what makes the named volume
# mounted at env/ come up writable: Docker seeds a fresh volume from the image
# directory, ownership included. A volume over a path that does not exist in the
# image would be root-owned instead, and the venv could never be created.
# config/ and config/pids are container-local, NOT bind-mounted from the host.
# bench only treats a directory as a bench when apps, sites, config, logs and
# config/pids all exist (bench.utils.paths_in_bench) — without them every
# site-aware command fails with "No such option: --site". The host's config/ is
# deliberately not reused: its redis confs hardcode host absolute paths for dir,
# pidfile and aclfile, and this container runs redis as separate services, so
# sharing it would also mean sharing pid files with the host bench.
RUN mkdir -p /workspace/bench/env /workspace/bench/logs /workspace/bench/config/pids \
    && chown -R "$UID:$GID" /workspace

COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh

USER frappe
ENV HOME=/home/frappe
WORKDIR /workspace/bench
ENTRYPOINT ["/usr/bin/tini", "--", "/usr/local/bin/entrypoint.sh"]
