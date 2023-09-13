FROM jenkins/jenkins:2.392-jdk11
ADD ./apache-maven-3.9.0-bin.tar.gz /usr/local/
ADD ./sonar-scanner-cli-4.8.0.2856-linux.tar.gz /usr/local/

USER root

WORKDIR /usr/local/
RUN mv sonar-scanner-4.8.0.2856-linux sonar-scanner-cli
RUN ln -s /usr/local/sonar-scanner-cli/bin/sonar-scanner /usr/bin/sonar-scanner

ENV MAVEN_HOME=/usr/local/apache-maven-3.9.0
ENV PATH=$JAVA_HOME/bin:$MAVEN_HOME/bin:$PATH

RUN echo "jenkins ALL=NOPASSWD: ALL" >> /etc/sudoers
USER jenkins
