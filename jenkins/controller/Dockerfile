FROM jenkins/jenkins:lts-jdk21

USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

USER jenkins
EXPOSE 8080
HEALTHCHECK --interval=15s --timeout=5s --start-period=90s --retries=20 \
    CMD curl --fail --silent http://127.0.0.1:8080/login || exit 1
