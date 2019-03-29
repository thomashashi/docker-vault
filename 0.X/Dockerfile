FROM centos:7

# This is the release of Vault to pull in.
ENV VAULT_VERSION=1.1.0

# These are the URL to the Vault Enterprise HSM Binary, checksums and signature
ENV HSMCLIENT_URL="https://s3.amazonaws.com/cloudhsmv2-software/CloudHsmClient/EL7/cloudhsm-client-latest.el7.x86_64.rpm"
ENV CHSM_URL="https://s3.amazonaws.com/cloudhsmv2-software/CloudHsmClient/EL7/cloudhsm-client-pkcs11-latest.el7.x86_64.rpm"
ENV VAULT_URL="https://s3-us-west-2.amazonaws.com/hc-enterprise-binaries/vault/ent.hsm/${VAULT_VERSION}/vault-enterprise_${VAULT_VERSION}%2Bent.hsm_linux_amd64.zip"
ENV VAULT_CHECKSUM="https://s3-us-west-2.amazonaws.com/hc-enterprise-binaries/vault/ent.hsm/${VAULT_VERSION}/vault-enterprise_${VAULT_VERSION}%2Bent.hsm_SHA256SUMS"
ENV VAULT_SIG="https://s3-us-west-2.amazonaws.com/hc-enterprise-binaries/vault/ent.hsm/${VAULT_VERSION}/vault-enterprise_${VAULT_VERSION}%2Bent.hsm_SHA256SUMS.sig"
ENV VAULT_GPGKEY=91A6E7F85D05C65630BEF18951852D87348FFC4C ARCH=amd64

# dumb-init is needed by docker-entrypoint.sh
ENV DUMB_INIT="https://github.com/Yelp/dumb-init/releases/download/v1.2.2/dumb-init_1.2.2_amd64"

# Create a vault user and group first so the IDs get set the same way,
# even as the rest of this may change over time.
RUN groupadd vault && \
    useradd --system --gid vault vault

# Set up certificates, our base tools, and Vault.
RUN set -eux; \
    yum -y install ca-certificates gnupg openssl libcap su-exec dumb-init tzdata libc6-compat libltdl unzip wget which && \
    found=''; \
    for server in \
        hkp://p80.pool.sks-keyservers.net:80 \
        hkp://keyserver.ubuntu.com:80 \
        hkp://pgp.mit.edu:80 \
    ; do \
        echo "Fetching GPG key $VAULT_GPGKEY from $server"; \
        gpg --batch --keyserver "$server" --recv-keys "$VAULT_GPGKEY" && found=yes && break; \
    done; \
    test -z "$found" && echo >&2 "error: failed to fetch GPG key $VAULT_GPGKEY" && exit 1; \
    mkdir -p /tmp/build && \
    cd /tmp/build && \
    wget ${HSMCLIENT_URL} &&\
    wget ${CHSM_URL} &&\
    wget ${VAULT_URL} && \
    wget ${VAULT_CHECKSUM} && \
    wget ${VAULT_SIG} && \
    yum install -y libtool-ltdl && \
    yum install -y ./cloudhsm-client-latest.el7.x86_64.rpm && \
    yum install -y ./cloudhsm-client-pkcs11-latest.el7.x86_64.rpm && \
    gpg --batch --verify vault-enterprise_${VAULT_VERSION}+ent.hsm_SHA256SUMS.sig vault-enterprise_${VAULT_VERSION}+ent.hsm_SHA256SUMS && \
    grep vault-enterprise_${VAULT_VERSION}+ent.hsm_linux_${ARCH}.zip vault-enterprise_${VAULT_VERSION}+ent.hsm_SHA256SUMS | sha256sum -c && \
    unzip vault-enterprise_${VAULT_VERSION}+ent.hsm_linux_${ARCH}.zip && \
    mv vault /bin/vault && \
    yum install -y python-setuptools && \
    easy_install supervisor && \
    wget -O dumb-init ${DUMB_INIT} && \
    mv dumb-init /usr/bin/dumb-init && \
    chmod +x /usr/bin/dumb-init && \
    cd /tmp && \
    rm -rf /tmp/build && \
    rm -rf /root/.gnupg


################################################################################

# /vault/logs is made available to use as a location to store audit logs, if
# desired; /vault/file is made available to use as a location with the file
# storage backend, if desired; the server will be started with /vault/config as
# the configuration directory so you can add additional config files in that
# location.
RUN mkdir -p /vault/logs && \
    mkdir -p /vault/file && \
    mkdir -p /vault/config && \
    chown -R vault:vault /vault

# Expose the logs directory as a volume since there's potentially long-running
# state in there
VOLUME /vault/logs

# Expose the file directory as a volume since there's potentially long-running
# state in there
VOLUME /vault/file

# 8200/tcp is the primary interface that applications use to interact with
# Vault.
EXPOSE 8200

# Expose /opt/cloudhsm/etc as a volume so you can mount in valid CloudHSM
# configuration
VOLUME /opt/cloudhsm/etc

# The entry point script uses dumb-init as the top-level process to reap any
# zombie processes created by Vault sub-processes.
# 
# The Enterprise+CloudHSM version uses supervisord, since you need to run both
# Vault and cloudhsm_client 
#
# For production derivatives of this container, you shoud add the IPC_LOCK
# capability so that Vault can mlock memory.
COPY vault_ent_cloudhsm.supervisord /etc/vault_ent_cloudhsm.supervisord
COPY docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["docker-entrypoint.sh"]

# By default you'll get a single-node development server that stores everything
# in RAM and bootstraps itself. Don't use this configuration for production.
CMD ["server", "-dev"]
