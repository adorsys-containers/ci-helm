FROM registry.access.redhat.com/ubi8/ubi:latest AS TZDATA

FROM openshift/origin-cli:v3.11 AS OC_CLIENT

# From https://github.com/containers/skopeo/blob/master/Dockerfile.build
FROM golang:1.14-buster as SKOPEO

ARG SKOPEO_VERSION=v1.2.0
RUN mkdir -p $GOPATH/src/github.com/containers/skopeo \
    && curl -LsSf https://github.com/containers/skopeo/archive/${SKOPEO_VERSION}.tar.gz | tar xz --strip-components=1 -C $GOPATH/src/github.com/containers/skopeo \
    && cd $GOPATH/src/github.com/containers/skopeo && make bin/skopeo DISABLE_CGO=1

FROM registry.access.redhat.com/ubi8/ubi-minimal:latest

LABEL maintainer="adorsys GmbH & Co. KG" \
      vendor="adorsys GmbH & Co. KG" \
      name="adorsys/ci-helm" \
      org.label-schema.vendor="adorsys GmbH & Co. KG" \
      org.label-schema.name="adorsys/ci-helm" \
      io.k8s.display-name="adorsys/ci-helm" \
      summary="adorsys/ci-helm" \
      io.k8s.description="adorsys/ci-helm" \
      org.label-schema.description="adorsys/ci-helm" \
      org.label-schema.schema-version="1.0" \
      org.label-schema.usage="" \
      org.label-schema.license="" \
      org.label-schema.build-date=""

# GNUPGHOME: Required von helm secrets
# Workaround for
#
# could not decrypt data key with PGP key: golang.org/x/crypto/openpgp error: Could not load secring: open ~/.gnupg/secring.gpg: no such file or directory;
#
# Since the current user does not have $HOME (in passwd) and ~ can resolve to the current home a PGP home must be defined.
# See also: https://github.com/mozilla/sops/blob/108e66f46f07a3c5dd0717d298f15558594baba6/pgp/keysource.go#L245
ENV GNUPGHOME=/var/local/lib/gnupg/ \
    TERM=xterm \
    HELM_HOME=/var/local/lib/helm/ \
    KUBECONFIG=/var/local/lib/kube/config \
    HOME=/tmp

ARG RUSH_VERSION=0.4.2
ARG SOPS_VERSION=v3.6.1
ARG JQ_VERSION=1.6
ARG YQ_VERSION=3.4.1
ARG HELM_VERSION=v2.17.0
ARG HELM_DIFF_VERSION=v2.11.0+5
ARG HELM_PUSH_VERSION=0.9.0
ARG HELM_SECRETS_VERSION=3.3.5
ARG KUBEVAL_VERSION=0.15.0

COPY root /

COPY --from=OC_CLIENT /usr/bin/oc /usr/local/bin/oc
COPY --from=SKOPEO /go/src/github.com/containers/skopeo/bin/skopeo /usr/local/bin/skopeo

# https://bugzilla.redhat.com/show_bug.cgi?id=1611117
COPY --from=TZDATA /usr/share/zoneinfo/UTC /usr/share/zoneinfo/UTC
COPY --from=TZDATA /usr/share/zoneinfo/Europe/Berlin /usr/share/zoneinfo/Europe/Berlin

RUN set -euo pipefail \
    && microdnf install --nodocs -y shadow-utils libsemanage \
    && useradd -d "${HOME}" -M -u 1001 -g 0 default \
    && microdnf remove -y shadow-utils libsemanage \
    && mkdir -p "${HELM_HOME}" "${GNUPGHOME}" "$(dirname "$KUBECONFIG")" \
    && echo -e '[docker-ce-stable]\nname=Docker CE Stable - $basearch\nbaseurl=https://download.docker.com/linux/centos/8/$basearch/stable\nenabled=0\ngpgcheck=1\ngpgkey=https://download.docker.com/linux/centos/gpg' > /etc/yum.repos.d/docker-ce.repo \
    && echo -e '[copr:copr.fedorainfracloud.org:jkroepke:git]\nname=Copr repo for git owned by jkroepke\nbaseurl=https://download.copr.fedorainfracloud.org/results/jkroepke/git/epel-8-$basearch/\ntype=rpm-md\nenabled=0\ngpgcheck=1\ngpgkey=https://download.copr.fedorainfracloud.org/results/jkroepke/git/pubkey.gpg\nrepo_gpgcheck=0\nenabled=1\nenabled_metadata=1' > /etc/yum.repos.d/jkroepke-git.repo \
    && microdnf update -y \
    && microdnf install --nodocs -y --enablerepo=copr:copr.fedorainfracloud.org:jkroepke:git --enablerepo=docker-ce-stable git-core docker-ce-cli tar unzip findutils sudo \
    && microdnf clean all \
## Configure sudo
    && printf 'Defaults !requiretty\nALL ALL=NOPASSWD: ALL' >> /etc/sudoers.d/all \
    && chmod 440 /etc/sudoers.d/all \
## Install yq
    && curl -LsSf https://github.com/stedolan/jq/releases/download/jq-${JQ_VERSION}/jq-linux64 -o /usr/local/bin/jq && chmod +x /usr/local/bin/jq \
    && curl -LsSf https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64 -o /usr/local/bin/yq && chmod +x /usr/local/bin/yq \
## Install rush
    && curl -LsSf https://github.com/shenwei356/rush/releases/download/v${RUSH_VERSION}/rush_linux_amd64.tar.gz | tar xz -C /usr/local/bin \
## Install kubernetes helm
    && curl -LsSf https://storage.googleapis.com/kubernetes-helm/helm-${HELM_VERSION}-linux-amd64.tar.gz | tar xz -C /usr/local/bin/ --strip-components=1 linux-amd64/helm  \
    && helm init --client-only \
    && helm repo remove local \
## Install helm plugins (manual installation without GH API)
    && mkdir -p "$(helm home)/plugins/helm-push" "$(helm home)/plugins/helm-kubeval/bin" \
    && curl -LsSf https://github.com/chartmuseum/helm-push/releases/download/v${HELM_PUSH_VERSION}/helm-push_${HELM_PUSH_VERSION}_linux_amd64.tar.gz | tar -C "$(helm home)/plugins/helm-push" -zxf- \
    && curl -LsSf https://github.com/databus23/helm-diff/releases/download/${HELM_DIFF_VERSION}/helm-diff-linux.tgz | tar -C "$(helm home)/plugins" --warning=no-unknown-keyword -zxf- \
    && curl -LsSf https://github.com/jkroepke/helm-secrets/releases/download/v${HELM_SECRETS_VERSION}/helm-secrets.tar.gz | tar -C "/var/local/lib/helm/plugins" -xzf- \
    && curl -LsSf https://github.com/mozilla/sops/releases/download/${SOPS_VERSION}/sops-${SOPS_VERSION}.linux -o /usr/local/bin/sops && chmod +x /usr/local/bin/sops \
    && curl -LsSf https://github.com/instrumenta/helm-kubeval/archive/master.tar.gz | tar --strip-components=1 -C "/var/local/lib/helm/plugins/helm-kubeval" -xzf- \
    && curl -LsSf https://github.com/instrumenta/kubeval/releases/download/${KUBEVAL_VERSION}/kubeval-linux-amd64.tar.gz | tar xz -C /usr/local/bin kubeval \
    && sed -i '/platformCommand:/,+2 d' /var/local/lib/helm/plugins/helm-secrets/plugin.yaml \
    && ln -s /usr/local/bin/kubeval /var/local/lib/helm/plugins/helm-kubeval/bin/ \
    && ln -s /usr/libexec/gpg-preset-passphrase /usr/local/bin/gpg-preset-passphrase \
    && chmod -R go+rw "$(helm home)" "${GNUPGHOME}" "$(dirname "$KUBECONFIG")" \
## Reduce image size
    && microdnf install --nodocs -y binutils \
    && strip /usr/local/bin/sops /usr/local/bin/skopeo /usr/local/bin/rush /usr/local/bin/oc /usr/local/bin/helm /usr/local/bin/kubeval \
    && microdnf remove -y binutils \
    && microdnf clean all \
    && rm -rf \
      /usr/libexec/docker/cli-plugins/docker-app \
      /var/cache/yum

USER 1001
