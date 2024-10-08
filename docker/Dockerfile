FROM python:3.12-alpine3.20

COPY docker/wait-for-it.sh /wait-for-it.sh
COPY requirements.txt requirements.txt
COPY docker/go.sum docker/go.mod /go/
ARG ADDITIONAL_REQUIREMENTS
# We run a single RUN command to decrease layer size - e.g. the final `apk del .build-deps` will
# remove some apk packages.
# Whois here is important - if we wouldn't install it, we would default to busybox whois,
# having different output making https://pypi.org/project/whoisdomain/ regexes fail.
RUN apk add --no-cache --virtual .build-deps go gcc git libc-dev make libffi-dev libpcap-dev postgresql-dev && \
    apk add --no-cache bash libpcap libpq git subversion whois && \
    cd /go && \
    GOBIN=/usr/local/bin/ go install -modfile go.mod github.com/projectdiscovery/naabu/v2/cmd/naabu && \
    GOBIN=/usr/local/bin/ go install -modfile go.mod github.com/praetorian-inc/fingerprintx/cmd/fingerprintx && \
    GOBIN=/usr/local/bin/ go install -modfile go.mod github.com/lc/gau/v2/cmd/gau && \
    GOBIN=/usr/local/bin/ go install -modfile go.mod github.com/projectdiscovery/subfinder/v2/cmd/subfinder && \
    GOBIN=/usr/local/bin/ go install -modfile go.mod github.com/projectdiscovery/nuclei/v3/cmd/nuclei && \
    cd / && \
    git clone https://github.com/rfc-st/humble.git --branch master /humble && \
    pip install --no-cache-dir -r requirements.txt -r /humble/requirements.txt $ADDITIONAL_REQUIREMENTS && \
    apk del .build-deps

WORKDIR /opt

COPY "artemis/" "artemis/"
COPY "alembic.ini" "alembic.ini"
COPY "migrations/" "migrations/"
COPY "static/" "static/"
COPY "templates/" "templates/"

# Warmup the cache. That way if the user is in an environment where requests to remote APIs (e.g. providing WordPress
# version check) fail, the response will be served from an on-disk cache.
RUN DB_CONN_STR=unused POSTGRES_CONN_STR=unused REDIS_CONN_STR=unused python3 -m artemis.fallback_api_cache

CMD ["bash", "-c", "alembic upgrade head && uvicorn artemis.main:app --host 0.0.0.0 --port 5000"]
