FROM python:3.12-slim-bookworm as builder

COPY requirements.txt /tmp
RUN \
    apt-get update && \
    apt-get install -y build-essential curl libpq-dev libmariadb-dev libffi-dev libssl-dev libcurl4-openssl-dev libsasl2-dev libldap2-dev libpython3-dev pkg-config
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | bash -s -- -y
ENV PATH=$PATH:/root/.cargo/bin
RUN \
    pip install --upgrade pip && \
    pip wheel --wheel-dir /wheels apprise uwsgi mysqlclient minio psycopg-c==3.2.1 -r /tmp/requirements.txt

COPY . /opt/healthchecks/
RUN \
    rm -rf /opt/healthchecks/.git && \
    rm -rf /opt/healthchecks/stuff

FROM python:3.12-slim-bookworm

RUN useradd --system hc
ENV PYTHONUNBUFFERED=1
WORKDIR /opt/healthchecks

RUN \
    apt-get update && \
    apt-get install -y ca-certificates libcurl4 libpq5 libmariadb3 libxml2 && \
    rm -rf /var/apt/cache && \
    rm -rf /var/lib/apt/lists


RUN --mount=type=bind,target=/wheels,source=/wheels,from=builder \
    pip install --upgrade pip && \
    pip install --no-cache /wheels/*

COPY --from=builder /opt/healthchecks/ /opt/healthchecks/
COPY docker/fetchstatus.py /opt/healthchecks/hc/api/management/commands/

RUN \
    rm -f /opt/healthchecks/hc/local_settings.py && \
    DEBUG=False SECRET_KEY=build-key ./manage.py collectstatic --noinput && \
    DEBUG=False SECRET_KEY=build-key ./manage.py compress

RUN mkdir /data && chown hc /data

USER hc

ENV USE_GZIP_MIDDLEWARE=True
HEALTHCHECK --interval=10s --start-period=10s --retries=1 CMD ./manage.py fetchstatus
CMD [ "uwsgi", "/opt/healthchecks/docker/uwsgi.ini"]
