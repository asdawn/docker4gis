# alpine 3.12 with tsinghua repos
# docker pull asdawn/cnqgis:3.12
FROM alpine:3.12

LABEL maintainer="dl@whu.edu.cn"
LABEL architect="all"
LABEL version="3.12"
LABEL description="Alpine with Tsinghua repos"

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories
RUN apk update
