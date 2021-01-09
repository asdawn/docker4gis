# ubuntu 20.04 with 163 repos
# docker pull asdawn/cnubuntu:20.10
FROM ubuntu:20.10

LABEL maintainer="dl@whu.edu.cn"
LABEL architect="all"
LABEL version="20.10"
LABEL description="Ubuntu with 163 repos"

# backup sources.list
RUN mv /etc/apt/sources.list /etc/apt/sources.list.bak
RUN echo "deb http://mirrors.163.com/ubuntu/ groovy main restricted universe multiverse" > /etc/apt/sources.list
RUN echo "deb http://mirrors.163.com/ubuntu/ groovy-security main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb http://mirrors.163.com/ubuntu/ groovy-updates main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb http://mirrors.163.com/ubuntu/ groovy-backports main restricted universe multiverse" >> /etc/apt/sources.list

RUN apt update
