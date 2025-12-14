FROM ubuntu:22.04

ARG USER_ID
ARG USER_NAME
ARG GROUP_ID
ARG GROUP_NAME
ARG WORK_DIR
ARG GO_VERSION

ENV DEBIAN_FRONTEND=noninteractive

# Base packages (libc6-dev for CGO)
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    tzdata \
    sudo \
    git \
    curl \
    openssh-client \
    jq \
    unzip \
    wget \
    make \
    gcc \
    libc6-dev \
    postgresql-client \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Set TimeZone
RUN ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime \
    && echo 'Asia/Tokyo' >/etc/timezone

# Create user and group
RUN groupadd --gid ${GROUP_ID} ${GROUP_NAME} \
    && useradd \
        --uid ${USER_ID} \
        --gid ${GROUP_ID} \
        --home-dir /home/${USER_NAME} \
        --create-home \
        --shell /bin/bash \
        ${USER_NAME} \
    && echo "${USER_NAME} ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers.d/${USER_NAME} \
    && chmod 0440 /etc/sudoers.d/${USER_NAME}

# Install Go
RUN ARCH=$(dpkg --print-architecture) \
    && curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-${ARCH}.tar.gz" -o go.tar.gz \
    && tar -C /usr/local -xzf go.tar.gz \
    && rm go.tar.gz

# Set Go environment variables
ENV GOROOT=/usr/local/go
ENV GOPATH=/home/${USER_NAME}/go
ENV PATH=${GOROOT}/bin:${GOPATH}/bin:${PATH}

# Setup vscode extensions cache directory
RUN mkdir -p /home/${USER_NAME}/.vscode-server/extensions \
    && chown -R ${USER_NAME}:${USER_NAME} /home/${USER_NAME}

USER ${USER_NAME}

# Install Go tools (pinned versions compatible with Go 1.23)
RUN go install github.com/air-verse/air@v1.61.7 \
    && go install github.com/go-delve/delve/cmd/dlv@v1.23.1

# Create working directory
RUN mkdir -p /home/${USER_NAME}/${WORK_DIR}

WORKDIR /home/${USER_NAME}/${WORK_DIR}
