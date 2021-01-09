# QGIS for geo-processing
# docker pull asdawn/cnqgis:3.16
FROM asdawn/cnubuntu:20.10

LABEL maintainer="dl@whu.edu.cn"
LABEL architect="all"
LABEL version="3.16"
LABEL description="QGIS for geo-processing"

RUN apt update
ARG  DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Hongkong
RUN apt install -y tzdata wget
# use qgis's repo to get the latest version
RUN apt install -y gnupg software-properties-common
RUN wget -qO - https://qgis.org/downloads/qgis-2020.gpg.key | gpg --no-default-keyring --keyring gnupg-ring:/etc/apt/trusted.gpg.d/qgis-archive.gpg --import
RUN chmod a+r /etc/apt/trusted.gpg.d/qgis-archive.gpg
RUN add-apt-repository "deb https://qgis.org/debian `lsb_release -c -s` main"
RUN apt update
RUN apt install -y qgis python3-qgis otb-qgis qgis-providers qgis-provider-grass qgis-plugin-grass libqgis-dev
