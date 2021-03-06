# ------------------------------------------------------------------------------
# Base image (for tests, precommit, codegen, etc.)
# ------------------------------------------------------------------------------
FROM ubuntu:xenial as base

# Install the runtime deps from apt
RUN apt-get update && \
    apt-get install -y curl make git gcc bzr unzip vim

# Install Golang 1.11
WORKDIR /usr/local
RUN curl https://dl.google.com/go/go1.11.linux-amd64.tar.gz -O && \
    tar xvf go1.11.linux-amd64.tar.gz && \
    cp -r go/bin/* /usr/local/bin/
ENV GO111MODULE on

# Env variables for specifying the output directories
ENV GOBIN /build/bin
ENV PLUGIN_DIR /build/plugins
ENV PATH ${PATH}:${GOBIN}

# Install protobuf compiler
RUN curl -Lfs https://github.com/google/protobuf/releases/download/v3.1.0/protoc-3.1.0-linux-x86_64.zip -o protoc3.zip && \
    unzip protoc3.zip -d protoc3 && \
    mv protoc3/bin/protoc /bin/protoc && \
    chmod a+rx /bin/protoc && \
    mv protoc3/include/google /usr/include/ && \
    chmod -R a+Xr /usr/include/google && \
    rm -rf protoc3.zip protoc3

# Copy just the go.mod files to download the golang deps from internet.
# This step would allow us to cache the downloads in a layer, and prevent
# reaching out to the internet unless any of the go.mod file is changed.
COPY gomod /magma
RUN for module in `ls -d /magma/**`; do cd ${module}/cloud/go && go mod download; done

# Copy the code
ENV MAGMA_ROOT /magma
COPY magma /magma
WORKDIR /magma/orc8r/cloud
RUN echo export MAGMA_MODULES="\"`ls -d /magma/** | tr '\n' ' '`\"" >> /etc/profile.d/env.sh
RUN . /etc/profile.d/env.sh && make tools

# ------------------------------------------------------------------------------
# Builder image with binaries and plugins
# ------------------------------------------------------------------------------
FROM base as builder

RUN . /etc/profile.d/env.sh && make build

# ------------------------------------------------------------------------------
# Production image
# ------------------------------------------------------------------------------
FROM ubuntu:xenial

# Install the runtime deps from apt
RUN apt-get update && \
    apt-get install -y supervisor daemontools openssl wget unzip

# Setup Swagger UI
COPY magma/orc8r/cloud/deploy/files/scripts/setup_swagger_ui /usr/local/bin/setup_swagger_ui
RUN /usr/local/bin/setup_swagger_ui

# Copy the build artifacts
COPY --from=builder /build/bin /var/opt/magma/bin
COPY --from=builder /build/plugins /var/opt/magma/plugins
COPY --from=builder /magma/orc8r/cloud/deploy/files/static/apidocs /var/opt/magma/static/apidocs

# Copy the scripts and configs from the context
COPY configs /etc/magma/configs
COPY magma/orc8r/cloud/deploy/files/envdir /var/opt/magma/envdir
COPY magma/orc8r/cloud/scripts/create_test_controller_certs /usr/local/bin/create_test_controller_certs

# Override the dev environments to suit Docker
# TODO: Remove this step after removing vagrant
RUN echo false > /var/opt/magma/envdir/SHOULD_INIT_METERING_TABLES && \
    echo "dbname=magma_dev user=magma_dev password=magma_dev host=datastore sslmode=disable" > /var/opt/magma/envdir/DATABASE_SOURCE

# Copy the supervisor configs
COPY magma/orc8r/cloud/deploy/roles/controller/files/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
CMD ["/usr/bin/supervisord"]