# Builder stage: install latest Claude Code binary and verify its integrity
# against the signed release manifest.
# This provides the newest features (e.g. /workflow, JS orchestration API)
# as "claude-unstable", while the RPM-installed "claude" stays on the stable
# channel for production reliability.
# Ref: https://code.claude.com/docs/en/setup#binary-integrity-and-code-signing
FROM registry.ci.openshift.org/ocp/builder:rhel-9-golang-1.25-openshift-4.22 AS claude-latest
RUN set -euo pipefail \
    # 1. Import the Anthropic release-signing GPG key
    && curl -fsSL https://downloads.claude.ai/keys/claude-code.asc | gpg --import \
    # 2. Install the latest Claude Code binary
    && curl -fsSL https://claude.ai/install.sh | bash -s latest \
    && test -x /root/.local/bin/claude \
    # 3. Determine installed version
    && CLAUDE_VERSION=$(/root/.local/bin/claude --version 2>&1 \
       | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1) \
    && echo "Installed claude version: ${CLAUDE_VERSION}" \
    # 4. Download the signed manifest for this version
    && RELEASES=https://downloads.claude.ai/claude-code-releases \
    && curl -fsSL -o /tmp/manifest.json \
       "${RELEASES}/${CLAUDE_VERSION}/manifest.json" \
    && curl -fsSL -o /tmp/manifest.json.sig \
       "${RELEASES}/${CLAUDE_VERSION}/manifest.json.sig" \
    # 5. Verify manifest GPG signature (fails build if invalid)
    && gpg --verify /tmp/manifest.json.sig /tmp/manifest.json \
    # 6. Verify binary SHA256 against the signed manifest
    && EXPECTED=$(python3 -c "                                    \
        import json, sys;                                         \
        m = json.load(open('/tmp/manifest.json'));                 \
        p = m.get('platforms', {});                                \
        e = p.get('linux-x64') or p.get('linux-x86_64') or {};    \
        cs = e.get('checksum', '');                                \
        cs = cs.removeprefix('sha256:');                           \
        print(cs) if cs else sys.exit('ERROR: no linux checksum') \
    ") \
    && ACTUAL=$(sha256sum /root/.local/bin/claude | awk '{print $1}') \
    && [ "${EXPECTED}" = "${ACTUAL}" ] \
    && echo "Binary integrity verified: checksum matches signed manifest"

FROM registry.ci.openshift.org/ocp/builder:rhel-9-golang-1.25-openshift-4.22

# Copy external rpm repos into the image, and tell the ART wrapper to append them
ENV ART_DNF_WRAPPER_POLICY=append
RUN rm /etc/yum.repos.d/ubi.repo
COPY images/repos/*.repo /etc/yum.repos.d/
COPY images/repos/RPM-GPG-KEY-* /etc/pki/rpm-gpg/

# Install all system packages in a single dnf transaction to avoid repeated
# metadata fetches (~800MB each). Disable foreign-arch repos to prevent
# cross-architecture dependency resolution failures in CI.
RUN dnf install -y \
    --disablerepo='*-ppc64le' --disablerepo='*-s390x' \
    python3.11 \
    python3.11-pip \
    python3.11-devel \
    gh \
    claude-code \
    pcp \
    pcp-devel \
    pcp-export-pcp2json \
    && dnf clean all \
    && alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 1 \
    && alternatives --set python3 /usr/bin/python3.11

# Copy the latest Claude Code binary from the builder stage as "claude-unstable".
# The claude binary is standalone, so it works without any additional dependencies.
COPY --from=claude-latest /root/.local/bin/claude /usr/local/bin/claude-unstable

# Install Python tools and PCP bindings
RUN python3 -m pip install --no-cache-dir \
    pytest \
    requests \
    pyyaml \
    && python3 -m pip install --no-cache-dir --use-pep517 pcp

# Create claude user with root group for OpenShift compatibility
RUN useradd -m -u 1000 -g 0 -s /bin/bash claude

# Copy ai-helpers repository to /opt/ai-helpers
COPY . /opt/ai-helpers
RUN chown -R claude:root /opt/ai-helpers

# Create Claude configuration directory and copy settings
#   Note: We pre-configure known_marketplaces.json because extraKnownMarketplaces doesn't seem to work in non-interactive mode.
RUN mkdir -p /home/claude/.claude/plugins
COPY images/claude-settings.json /home/claude/.claude/settings.json
COPY images/known_marketplaces.json /home/claude/.claude/plugins/known_marketplaces.json
RUN chown -R claude:root /home/claude

# Create workspace directory
RUN mkdir -p /workspace && chown -R claude:root /workspace
WORKDIR /workspace

# Switch to claude user
USER claude

# In CI, OpenShift uses a random UID, but in the root group
RUN chmod -R g+rwx /home/claude && \
  chmod -R g+rwx /opt/ai-helpers && \
  chmod -R g+rwx /workspace

ENV CLAUDE_CONFIG_DIR="/home/claude/.claude"

# Set Claude Code as the default command
ENTRYPOINT ["claude"]

# Default to showing help
CMD ["--help"]
