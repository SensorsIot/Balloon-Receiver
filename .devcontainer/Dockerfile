FROM mcr.microsoft.com/devcontainers/base:debian

ARG IDF_BRANCH=release/v5.5
ENV IDF_PATH=/opt/esp-idf
ENV IDF_TOOLS_PATH=/opt/esp-idf-tools

# Rename vscode user to dev (base image has vscode with UID 1000)
RUN usermod -l dev -d /home/dev -m vscode \
    && groupmod -n dev vscode \
    && sed -i 's/vscode/dev/g' /etc/sudoers.d/vscode || true \
    && mv /etc/sudoers.d/vscode /etc/sudoers.d/dev 2>/dev/null || true

# Remove unused zsh/oh-my-zsh
RUN rm -rf /home/*/.oh-my-zsh /root/.oh-my-zsh && apt-get purge -y zsh && apt-get autoremove -y

# ESP-IDF build dependencies, Python, USB tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    git wget flex bison gperf \
    cmake ninja-build ccache \
    libffi-dev libssl-dev dfu-util libusb-1.0-0 \
    python3 python3-pip python3-venv python3-dev \
    build-essential \
    udev usbutils minicom screen \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# udev rules for ESP32 USB serial adapters (Espressif, CP210x, CH340)
RUN mkdir -p /etc/udev/rules.d \
    && printf '%s\n' \
        'SUBSYSTEM=="tty", ATTRS{idVendor}=="303a", MODE="0666"' \
        'SUBSYSTEM=="tty", ATTRS{idVendor}=="10c4", MODE="0666"' \
        'SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", MODE="0666"' \
        'SUBSYSTEM=="tty", ATTRS{idVendor}=="0403", MODE="0666"' \
        > /etc/udev/rules.d/99-esp32.rules

# Install ESP-IDF (clone + tools install)
RUN mkdir -p ${IDF_PATH} ${IDF_TOOLS_PATH} \
    && git clone -b ${IDF_BRANCH} --recursive --depth 1 --shallow-submodules \
        https://github.com/espressif/esp-idf.git ${IDF_PATH} \
    && ${IDF_PATH}/install.sh all \
    && chown -R dev:dev ${IDF_PATH} ${IDF_TOOLS_PATH}

# Add dev to dialout for serial access
RUN usermod -aG dialout dev

# Source ESP-IDF in dev's bashrc
RUN echo '. /opt/esp-idf/export.sh > /dev/null 2>&1' >> /home/dev/.bashrc

# SSH quiet config
RUN mkdir -p /home/dev/.ssh \
    && printf 'Host *\n    StrictHostKeyChecking accept-new\n    LogLevel ERROR\n' > /home/dev/.ssh/config \
    && chmod 700 /home/dev/.ssh \
    && chmod 600 /home/dev/.ssh/config \
    && chown -R dev:dev /home/dev/.ssh

# Auto-cd to workspace for SSH sessions
RUN echo 'cd /workspaces/Balloon-Receiver 2>/dev/null' >> /home/dev/.bashrc

WORKDIR /workspaces/Balloon-Receiver
USER dev

# tmux — terminal multiplexer for SSH sessions
USER root
RUN apt-get update && apt-get install -y --no-install-recommends tmux \
    && apt-get clean && rm -rf /var/lib/apt/lists/*
USER dev

# Per-user npm prefix on PATH so claude (installed into ~/.npm-global/bin by
# postCreate) and the superclaude alias resolve. Set as image ENV, not appended
# to .bashrc: Debian's .bashrc returns early for non-interactive shells, so a
# PATH export there is invisible to `ssh <host> claude ...`, cron and scripts.
# ENV applies to every shell, interactive or not.
ENV PATH="/home/dev/.npm-global/bin:${PATH}"

# Force UTF-8 locale so tmux renders Unicode block glyphs (claude's splash logo)
# correctly. Must also come before the tmux guard.
RUN echo 'export LANG=C.UTF-8' >> /home/dev/.bashrc

# Persistent claude session: interactive SSH logins auto-attach a tmux session
# "main" running claude, so it survives disconnects (reconnect = continue).
RUN printf '%s\n' \
    '' \
    '# --- Persistent claude session (auto-attach tmux on interactive SSH) ---' \
    'if [ -n "$SSH_CONNECTION" ] && [ -z "$TMUX" ] && [ "$TERM_PROGRAM" != "vscode" ] && [ ! -f "$HOME/.no-tmux" ] && command -v tmux >/dev/null 2>&1; then' \
    '    case "$-" in' \
    '        *i*) exec tmux new-session -A -s main -c /workspaces/Balloon-Receiver "claude --dangerously-skip-permissions; exec bash" ;;' \
    '    esac' \
    'fi' \
    >> /home/dev/.bashrc
