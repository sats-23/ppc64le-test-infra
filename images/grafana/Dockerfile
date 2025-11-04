# Build stage
FROM registry.access.redhat.com/ubi9/ubi:9.6 AS builder

RUN dnf update -y && \
    dnf install -y --allowerasing\
        git curl wget ca-certificates python3 python3-pip make && \
    dnf clean all && \
    rm -rf /var/cache/dnf

#Install Go 
ARG GO_VERSION="1.25.0"

RUN wget https://go.dev/dl/go${GO_VERSION}.$(echo "$(uname -s)" | tr A-Z a-z)-$(uname -m).tar.gz && tar -xvzf go${GO_VERSION}.$(echo "$(uname -s)" | tr A-Z a-z)-$(uname -m).tar.gz -C /usr/local 

ENV GOPATH=/usr/local/go
ENV PATH=$PATH:/usr/local/go/bin

#Install Node.js
ENV NVM_DIR=/root/.nvm
ENV NODE_VERSION=22.16.0

RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash && \
    . "$NVM_DIR/nvm.sh" && \
    nvm install $NODE_VERSION && \
    nvm use $NODE_VERSION && \
    nvm alias default $NODE_VERSION

ENV PATH=$NVM_DIR/versions/node/v$NODE_VERSION/bin:$PATH
ENV NODE_OPTIONS=--max_old_space_size=12000

RUN corepack enable && corepack prepare yarn@stable --activate

WORKDIR /src

COPY 0001-patch-swc-loader-to-babel-for-ppc64le.patch /tmp/

RUN git clone https://github.com/grafana/grafana.git . && \
    git checkout v12.1.0 && \
    git apply /tmp/0001-patch-swc-loader-to-babel-for-ppc64le.patch && \
    make deps-go  && \
    make deps  && \
    yarn add --dev babel-loader @babel/core @babel/preset-env @babel/preset-typescript @babel/preset-react && \
    make build

# Runtime stage
FROM registry.access.redhat.com/ubi9/ubi:9.6

RUN dnf update -y && \
    dnf install -y --setopt=install_weak_deps=false \
    ca-certificates curl tini && \
    dnf clean all && \
    rm -rf /var/cache/dnf

RUN useradd -u 472 -r -g 0 -d /usr/share/grafana -s /sbin/nologin grafana

WORKDIR /usr/share/grafana

COPY --from=builder /src/bin/linux-ppc64le/grafana ./bin/
COPY --from=builder /src/conf ./conf
COPY --from=builder /src/public ./public

ENV PATH=/usr/share/grafana/bin:$PATH

RUN chown -R grafana:root /usr/share/grafana && chmod -R g+rwX /usr/share/grafana

USER grafana

EXPOSE 3000

ENTRYPOINT ["/usr/share/grafana/bin/grafana"]
CMD ["server", "--homepath=/usr/share/grafana"]
