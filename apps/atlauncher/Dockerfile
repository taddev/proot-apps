ARG REPO
FROM ghcr.io/${REPO}:debian-bookworm AS rootfs
ARG REPO

# Install packages
RUN \
  echo "**** install packages ****" && \
  apt-get update && \
  apt-get install -y --no-install-recommends \
    default-jdk \
    wget \
    desktop-file-utils && \
  echo "**** download at-launcher ****" && \
  curl -o \
    /tmp/atluancher.deb -L \
    "https://atlauncher.com/download/deb" && \
  dpkg -i /tmp/atlauncher.deb && \
  echo "**** desktop file ****" && \
  DESKTOP_FILE=/usr/share/applications/atlauncher.desktop && \
  echo '[Desktop Entry]' > ${DESKTOP_FILE} && \
  echo 'Type=Application' >> ${DESKTOP_FILE} && \
  echo 'Name=ATLauncher PA' >> ${DESKTOP_FILE} && \
  echo 'Icon=atlauncher' >> ${DESKTOP_FILE} && \
  echo 'Exec=/bin/sh -c "$HOME/.local/bin/proot-apps run ghcr.io/'${REPO}':atlauncher %U"' >> ${DESKTOP_FILE} && \
  echo "**** set bin name ****" && \
  echo "atlauncher-pa" > /bin-name && \
  echo "**** cleanup ****" && \
  apt-get autoclean && \
  rm -rf \
    /root/.cache \
    /var/lib/apt/lists/* \
    /var/tmp/* \
    /tmp/*

COPY ./entrypoint /entrypoint
COPY ./install /install
COPY ./remove /remove

FROM scratch
COPY --from=rootfs / /
