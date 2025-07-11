FROM jenkins/jenkins:lts

USER root

# Install Docker CLI (optional, but useful for pipeline Docker commands)
RUN apt-get update && \
    apt-get install -y docker.io && \
    usermod -aG docker jenkins

USER jenkins
