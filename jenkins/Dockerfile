FROM jenkins/jenkins:lts-jdk21

USER root

RUN apt-get update \
    && apt-get install -y ca-certificates curl git python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*

RUN curl -fsSL https://get.docker.com | sh

RUN curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
    | sh -s -- -b /usr/local/bin

USER jenkins
