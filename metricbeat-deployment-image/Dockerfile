FROM registry.access.redhat.com/ubi8/ubi:latest

RUN rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch 

COPY ./elastic.repo /etc/yum.repos.d/elastic.repo

RUN yum install -y metricbeat &&\
    systemctl enable metricbeat

RUN chmod -R 777 /etc/metricbeat &&\
    chmod go-w /etc/metricbeat/modules.d/system.yml &&\
    mkdir -p /var/lib/metricbeat &&\
    chmod -R 777 /var/lib/metricbeat &&\
    chmod -R 777 /var/log

