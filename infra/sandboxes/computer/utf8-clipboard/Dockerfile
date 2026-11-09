ARG RAKAZO_COMPUTER_IMAGE=ghcr.io/elie222/rakazo/computer:sha-46e95fa8314324cbcf6e4e33c7ed06dbbc68b923
FROM ${RAKAZO_COMPUTER_IMAGE} AS clipboard-builder
USER root
ENV LD_PRELOAD=""
RUN sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/debian.sources \
  && apt-get update \
  && apt-get install -y --no-install-recommends build-essential dpkg-dev patch \
  && apt-get build-dep -y --no-install-recommends x11vnc
WORKDIR /build
RUN apt-get source "x11vnc=$(dpkg-query -W -f='${source:Version}' x11vnc)"
COPY x11vnc-utf8.patch novnc-provide.patch /build/
RUN cd /build/x11vnc-* \
  && patch -p1 < /build/x11vnc-utf8.patch \
  && dpkg-buildpackage -b -uc -us -j4 \
  && dpkg-deb --extract /build/x11vnc_*.deb /build/install \
  && cd /usr/share/novnc \
  && patch -p1 < /build/novnc-provide.patch
FROM ${RAKAZO_COMPUTER_IMAGE} AS computer-clipboard
COPY --from=clipboard-builder /build/install/usr/bin/x11vnc /usr/bin/x11vnc
COPY --from=clipboard-builder /usr/share/novnc/core/rfb.js /usr/share/novnc/core/rfb.js
# Xvfb has no login manager whose cleanup could destroy selection ownership.
ENV X11VNC_AVOID_WINDOWS=never
