# Official PostgreSQL 16 image (Debian-based), plus Patroni in a virtualenv.
# A venv is used because Debian's system Python blocks global pip installs.
FROM postgres:16

USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 python3-venv \
    && python3 -m venv /opt/patroni \
    && /opt/patroni/bin/pip install --no-cache-dir "patroni[etcd3]" psycopg2-binary \
    && rm -rf /var/lib/apt/lists/*

COPY entrypoint.sh /usr/local/bin/entrypoint-patroni.sh
RUN chmod +x /usr/local/bin/entrypoint-patroni.sh

USER postgres
ENTRYPOINT ["/usr/local/bin/entrypoint-patroni.sh"]
