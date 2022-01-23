FROM haproxy:1.7
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
RUN mkdir /run/haproxy &&\
    apt-get update -y &&\
    apt-get install -y hatop &&\
    apt-get clean
	