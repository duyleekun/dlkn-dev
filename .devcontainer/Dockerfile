FROM ubuntu:24.04

# Install sudo, locales, and create user
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
--mount=type=cache,target=/var/lib/apt,sharing=locked apt update && apt install -y sudo curl git build-essential nano vim locales libonig-dev && \
    locale-gen en_US.UTF-8 && \
    update-locale LANG=en_US.UTF-8

# Set locale environment variables for Ansible
ENV LANG=en_US.UTF-8
ENV LC_ALL=en_US.UTF-8

ARG USER_NAME
ARG USER_UID
ARG USER_GID


RUN groupadd -g ${USER_GID} ${USER_NAME} || echo "Group ${USER_NAME} already exists"
RUN useradd -m -u ${USER_UID} -g ${USER_GID} -G sudo ${USER_NAME} || echo "User ${USER_NAME} already exists"
RUN echo "${USER_NAME} ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers 

USER ${USER_NAME}
WORKDIR /home/${USER_NAME}

# Set Homebrew environment variables before installation
ENV HOMEBREW_PREFIX="/home/linuxbrew/.linuxbrew"
ENV HOMEBREW_CELLAR="/home/linuxbrew/.linuxbrew/Cellar"
ENV HOMEBREW_REPOSITORY="/home/linuxbrew/.linuxbrew/Homebrew"
ENV PATH="/home/linuxbrew/.linuxbrew/bin:/home/linuxbrew/.linuxbrew/sbin:$PATH"
ENV INFOPATH="/home/linuxbrew/.linuxbrew/share/info"
ENV HOMEBREW_CACHE="/home/${USER_NAME}/.cache/Homebrew"
# Create cache directory for Homebrew
RUN mkdir -p $HOMEBREW_CACHE && \
    chmod 755 $HOMEBREW_CACHE

# install linux brew
RUN --mount=type=cache,target=/home/${USER_NAME}/.cache/Homebrew \
    NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)" && \
    brew update && \
    brew install kubectl helm uv k9s
    
# install python 3.11
ENV VIRTUAL_ENV=/home/${USER_NAME}/.venv
RUN uv venv --python 3.11
RUN uv pip install ansible jinja2 netaddr pyyaml ansible-dev-tools