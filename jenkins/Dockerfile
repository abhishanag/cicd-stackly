FROM jenkins/jenkins:lts-jdk21

USER root

RUN apt-get update && \
    apt-get install -y docker.io python3 python3-pip && \
    rm -rf /var/lib/apt/lists/*

RUN DOCKER_GROUP=$(getent group 990 | cut -d: -f1) && \
    if [ -z "$DOCKER_GROUP" ]; then \
        groupadd -g 990 docker; \
        DOCKER_GROUP=docker; \
    fi && \
    usermod -aG "$DOCKER_GROUP" jenkins

RUN jenkins-plugin-cli --plugins workflow-aggregator git credentials-binding

USER jenkins
