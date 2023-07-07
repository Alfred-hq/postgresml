FROM nvidia/cuda:12.1.1-devel-ubuntu22.04
LABEL maintainer="team@postgresml.com"

ARG DEBIAN_FRONTEND=noninteractive
ARG PGML_VERSION=2.7.0
ENV TZ=Etc/UTC
# ENV PATH="/usr/local/cuda/bin:${PATH}"

# explicitly set user/group IDs
RUN set -eux; \
	groupadd -r postgres --gid=999; \
# https://salsa.debian.org/postgresql/postgresql-common/blob/997d842ee744687d99a2b2d95c1083a2615c79e8/debian/postgresql-common.postinst#L32-35
	useradd -r -g postgres --uid=999 --home-dir=/var/lib/postgresql --shell=/bin/bash postgres; \
# also create the postgres user's home directory with appropriate permissions
# see https://github.com/docker-library/postgres/issues/274
	mkdir -p /var/lib/postgresql; \
	chown -R postgres:postgres /var/lib/postgresql

RUN set -ex; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		gnupg \
	; \
	rm -rf /var/lib/apt/lists/*

# Dependencies
RUN apt-get update && \
    apt-get install -y \
    curl \
    lsb-release \
    python3 \
    python3-pip \
    tzdata \
    sudo \
    cmake \
    libpq-dev \
    libclang-dev \
    wget \
    git \
    gnupg \
    lsb-release

RUN set -eux; \
	if [ -f /etc/dpkg/dpkg.cfg.d/docker ]; then \
# if this file exists, we're likely in "debian:xxx-slim", and locales are thus being excluded so we need to remove that exclusion (since we need locales)
		grep -q '/usr/share/locale' /etc/dpkg/dpkg.cfg.d/docker; \
		sed -ri '/\/usr\/share\/locale/d' /etc/dpkg/dpkg.cfg.d/docker; \
		! grep -q '/usr/share/locale' /etc/dpkg/dpkg.cfg.d/docker; \
	fi; \
	apt-get update; apt-get install -y --no-install-recommends locales; rm -rf /var/lib/apt/lists/*; \
	localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG en_US.utf8

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		libnss-wrapper \
		xz-utils \
		zstd \
	; \
	rm -rf /var/lib/apt/lists/*

RUN mkdir /docker-entrypoint-initdb.d

ENV PG_MAJOR 14
ENV PATH $PATH:/usr/lib/postgresql/$PG_MAJOR/bin

ENV PG_VERSION 14.8-1.pgdg110+1

# PostgreSQL
RUN wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && \
    echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list && \
    apt-get update && \
    apt-get install -y postgresql-14 && \
    apt-get install -y postgresql-plpython3-14

RUN	apt-get install -y --no-install-recommends postgresql-common; \
	sed -ri 's/#(create_main_cluster) .*$/\1 = false/' /etc/postgresql-common/createcluster.conf;

RUN postgres --version


# make the sample config easier to munge (and "correct by default")
RUN set -eux; \
	dpkg-divert --add --rename --divert "/usr/share/postgresql/postgresql.conf.sample.dpkg" "/usr/share/postgresql/$PG_MAJOR/postgresql.conf.sample"; \
	cp -v /usr/share/postgresql/postgresql.conf.sample.dpkg /usr/share/postgresql/postgresql.conf.sample; \
	ln -sv ../postgresql.conf.sample "/usr/share/postgresql/$PG_MAJOR/"; \
	sed -ri "s!^#?(listen_addresses)\s*=\s*\S+.*!\1 = '*'!" /usr/share/postgresql/postgresql.conf.sample; \
	grep -F "listen_addresses = '*'" /usr/share/postgresql/postgresql.conf.sample

RUN echo "shared_preload_libraries = 'pgml'" > /usr/share/postgresql/postgresql.conf.sample

RUN mkdir -p /var/run/postgresql && chown -R postgres:postgres /var/run/postgresql && chmod 2777 /var/run/postgresql

ENV PGDATA /var/lib/postgresql/data
# this 777 will be replaced by 700 at runtime (allows semi-arbitrary "--user" values)
RUN mkdir -p "$PGDATA" && chown -R postgres:postgres "$PGDATA" && chmod 777 "$PGDATA"
VOLUME /var/lib/postgresql/data

COPY docker/docker-entrypoint.sh /usr/local/bin/

STOPSIGNAL SIGINT

# PostgresML
RUN echo "deb [trusted=yes] https://apt.postgresml.org $(lsb_release -cs) main" >> /etc/apt/sources.list
RUN cat /etc/apt/sources.list
RUN apt-get update && apt-get install -y postgresql-pgml-14=${PGML_VERSION}

COPY requirements.txt /app/requirements.txt
WORKDIR /app
RUN pip3 install -r requirements.txt

COPY requirements-xformers.txt /app/requirements-xformers.txt
RUN pip3 install -r requirements-xformers.txt --no-dependencies

COPY . /app
# Listen on 0.0.0.0 and allow 'root' to connect without a password.
# Please modify for production deployments accordingly.
RUN cp /app/docker/postgresql.conf /etc/postgresql/14/main/postgresql.conf
RUN cp /app/docker/pg_hba.conf /etc/postgresql/14/main/pg_hba.conf
RUN cd /tmp && \
    git clone --branch v0.4.1 https://github.com/pgvector/pgvector.git && \
    cd pgvector && \
    make && \
    make install

ENTRYPOINT ["/bin/bash", "/app/docker/entrypoint.sh"]