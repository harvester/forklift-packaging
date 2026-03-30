FROM registry.suse.com/bci/bci-base:15.7
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN zypper -n --gpg-auto-import-keys refresh && \
    zypper -n install bash awk && \
    zypper clean -a
RUN curl -sOL https://github.com/open-policy-agent/opa/releases/download/v0.62.1/opa_linux_amd64 && \
    curl -sOL https://github.com/open-policy-agent/opa/releases/download/v0.62.1/opa_linux_amd64.sha256 && \
    read -r EXPECTED_CHECKSUM < <(cat opa_linux_amd64.sha256 | awk '{print $1}') && \
    read -r ACTUAL_CHECKSUM < <(sha256sum opa_linux_amd64 | awk '{print $1}') && \
    if [ "$EXPECTED_CHECKSUM" != "$ACTUAL_CHECKSUM" ]; then \
       echo "Checksum verification failed!" && \
        exit 1; \         
    fi && \
    mv opa_linux_amd64 /usr/bin/opa
RUN chmod +x /usr/bin/opa
COPY validation/policies /usr/share/opa/policies/
COPY validation/entrypoint.sh /usr/bin/

ENTRYPOINT ["/usr/bin/entrypoint.sh"]