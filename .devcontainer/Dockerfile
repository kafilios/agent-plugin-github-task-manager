FROM mcr.microsoft.com/devcontainers/base:ubuntu

ARG DEBIAN_FRONTEND=noninteractive
ENV DEBIAN_FRONTEND=noninteractive

# bubblewrap setuid required for namespace sandboxes on restricted hosts.
RUN apt-get update \
  && apt-get install -y --no-install-recommends ca-certificates curl bubblewrap libcap2-bin \
  && bwrap="$(command -v bwrap)" \
  && chown root:root "$bwrap" && chmod 4755 "$bwrap" \
  && rm -rf /var/lib/apt/lists/*

RUN mkdir -p /home/linuxbrew/.linuxbrew && chown -R vscode:vscode /home/linuxbrew
ENV NONINTERACTIVE=1
USER vscode
RUN /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)" \
  && /home/linuxbrew/.linuxbrew/bin/brew --version \
  && echo >> /home/vscode/.bashrc \
  && echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv bash)"' >> /home/vscode/.bashrc \
  && eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv bash)" \
  && /home/linuxbrew/.linuxbrew/bin/brew install 1password-cli betterleaks claude-code composer curl dbxcli docker docker-buildx docker-compose gh git-secret gnupg jq node pandoc php socat wget yq \
  && docker context create exit-au-oci --docker "host=ssh://docker@exit-au-oci" \
  && mkdir -p /home/vscode/.docker/cli-plugins \
  && ln -sf /home/linuxbrew/.linuxbrew/bin/docker-compose /home/vscode/.docker/cli-plugins/docker-compose \
  && ln -sf /home/linuxbrew/.linuxbrew/bin/docker-buildx /home/vscode/.docker/cli-plugins/docker-buildx
USER root

RUN set -eux; \
  export PATH="/home/linuxbrew/.linuxbrew/bin:$PATH"; \
  npm install -g @anthropic-ai/sandbox-runtime; \
  npm cache clean --force; \
  mkdir -p /usr/local/share/sandbox-runtime/seccomp; \
  src="$(npm root -g)/@anthropic-ai/sandbox-runtime/vendor/seccomp"; \
  test -d "$src"; \
  cp -a "$src/." /usr/local/share/sandbox-runtime/seccomp/; \
  chmod -R a+rX /usr/local/share/sandbox-runtime/seccomp;

ENV EDITOR="code --wait"
ENV BROWSER=code
ENV GNUPGHOME=/home/vscode/.gnupg-container

# Pin a fixed local SSH socket so the container never inherits a host-forwarded
# agent (VS Code / DevPod). Bootstrap (this repo's script/bootstrap, invoked
# via the npx-gated 99-bootstrap_dotfiles.sh in this directory) is expected
# to start ssh-agent on this path and stream the auth key from 1Password.
ENV SSH_AUTH_SOCK=/tmp/ssh-agent.sock

COPY etc/gitconfig /etc/gitconfig
RUN mkdir -p /home/vscode/.ssh && chown vscode:vscode /home/vscode/.ssh && chmod 700 /home/vscode/.ssh
COPY --chmod=644 --chown=vscode:vscode config/.ssh/known_hosts /home/vscode/.ssh/known_hosts
COPY --chmod=755 bin/postAttachCommand.sh /usr/local/bin/postAttachCommand.sh
COPY --chmod=755 bin/postAttachCommand.d/ /usr/local/bin/postAttachCommand.d/
