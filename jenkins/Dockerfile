FROM jenkins/jenkins:lts-jdk21

USER root

RUN apt-get update \
    && apt-get install -y docker.io git curl vim ca-certificates \
    && rm -rf /var/lib/apt/lists/*

USER root
