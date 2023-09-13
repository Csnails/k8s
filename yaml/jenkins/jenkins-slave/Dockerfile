FROM centos:7

RUN yum install -y java-1.8.0-openjdk curl git libtool-ltdl-devel && yum clean all && rm -rf /var/cache/yum/* && mkdir -p /var/jenkins_home

COPY agent.jar /var/jenkins_home/slave.jar
COPY jenkins-agent /usr/bin/jenkins-agent
COPY apache-maven-3.9.0-bin.tar.gz /usr/local/
COPY settings.xml /usr/local/apache-maven-3.9.0

RUN chmod +x /usr/bin/jenkins-agent

ENTRYPOINT ["jenkins-agent"]
