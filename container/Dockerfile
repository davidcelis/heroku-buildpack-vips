ARG STACK_VERSION
FROM heroku/heroku:$STACK_VERSION

ENV DEBIAN_FRONTEND noninteractive

# generic build tools  ... libgsf needs intltool
RUN apt-get update \
  && apt-get install -y \
  build-essential \
  wget \
  python3-pip \
  ninja-build \
  pkg-config

RUN pip3 install meson

# use the heroku platform libraries when we can
#
# see https://devcenter.heroku.com/articles/stack-packages
#
# libgsf needs libxml2
#
# this should only pull in header files and should not create any extra run
# time dependencies
RUN apt-get install -y \
  glib-2.0-dev \
  libexpat1-dev \
  libpango1.0-dev \
  librsvg2-dev \
  libpng-dev \
  libwebp-dev \
  libjpeg-turbo8-dev \
  libtiff5-dev \
  libexif-dev \
  liblcms2-dev \
  libgsf-1-dev \
  liborc-0.4-dev \
  libimagequant-dev \
  libcgif-dev \
  libheif-dev \
  libfftw3-dev

WORKDIR /usr/local/src

# build to this prefix
#  - heroku has /usr/local/lib on the default ld.so.conf search path, so
#    this is convenient
#  - heroku has a basic dir structure in /usr/local, but no files
ARG PREFIX=/usr/local
ENV PKG_CONFIG_PATH=$PREFIX/lib/pkgconfig

# last updated 16sep23
ARG PDFIUM_VERSION=6002
ARG PDFIUM_URL=https://github.com/bblanchon/pdfium-binaries/releases/download/chromium

RUN wget $PDFIUM_URL/$PDFIUM_VERSION/pdfium-linux-x64.tgz \
  && mkdir pdfium \
  && cd pdfium \
  && tar xf ../pdfium-linux-x64.tgz \
  && cp lib/* $PREFIX/lib \
  && cp -r include/* $PREFIX/include

# make a pdfium.pc file libvips can use
RUN mkdir -p $PREFIX/lib/pkgconfig \
  && cd $PREFIX/lib/pkgconfig \
  && echo "prefix=$PREFIX" >> pdfium.pc \
  && echo "exec_prefix=\${prefix}" >> pdfium.pc \
  && echo "libdir=\${exec_prefix}/lib" >> pdfium.pc \
  && echo "includedir=\${prefix}/include" >> pdfium.pc \
  && echo "Name: pdfium" >> pdfium.pc \
  && echo "Description: pdfium" >> pdfium.pc \
  && echo "Version: $PDFIUM_VERSION" >> pdfium.pc \
  && echo "Requires: " >> pdfium.pc \
  && echo "Libs: -L\${libdir} -lpdfium" >> pdfium.pc \
  && echo "Cflags: -I\${includedir}" >> pdfium.pc

# use libspng for PNG load and save
ARG SPNG_VERSION=0.7.4
ARG SPNG_URL=https://github.com/randy408/libspng/archive/refs/tags

RUN wget ${SPNG_URL}/v${SPNG_VERSION}.tar.gz \
  && tar xf v${SPNG_VERSION}.tar.gz \
  && cd libspng-${SPNG_VERSION} \
  && meson build --prefix=${PREFIX} --libdir=lib \
  && cd build \
  && ninja \
  && ninja install

ARG VIPS_VERSION=8.14.4
ARG VIPS_URL=https://github.com/libvips/libvips/releases/download

RUN wget ${VIPS_URL}/v${VIPS_VERSION}/vips-${VIPS_VERSION}.tar.xz \
  && tar xf vips-${VIPS_VERSION}.tar.xz \
  && cd vips-${VIPS_VERSION} \
  && meson setup build \
  --prefix=$PREFIX \
  --libdir=lib \
  --buildtype=release \
  -Dradiance=false \
  -Danalyze=false \
  -Dintrospection=false \
  && cd build \
  && meson compile \
  && meson install

# clean and package
ARG STACK_VERSION
RUN cd $PREFIX \
  && rm -rf bin/gif* bin/orc* bin/gsf* bin/batch_* bin/vips-* \
  && rm -rf bin/vipsprofile bin/light_correct bin/shrink_width \
  && rm -rf lib/*.a lib/*.la lib/python* \
  && strip lib/lib*.so* \
  && rm -rf build \
  && mkdir -p build \
  && tar czf "build/heroku-${STACK_VERSION}.tar.gz" bin include lib \
  && ls -l build/heroku-${STACK_VERSION}.tar.gz \
  && echo built heroku-${STACK_VERSION}.tar.gz

# Update ldconfig cache
RUN ldconfig

# Store configuration for easy viewing in the repo
ARG STACK_VERSION
RUN cd $PREFIX \
  && echo "$ vips --vips-version" > "build/heroku-$STACK_VERSION.config.log" \
  && vips --vips-version >> "build/heroku-$STACK_VERSION.config.log" \
  && echo "" >> "build/heroku-$STACK_VERSION.config.log" \
  && echo "$ vips --vips-config" >> "build/heroku-$STACK_VERSION.config.log" \
  && vips --vips-config >> "build/heroku-$STACK_VERSION.config.log"

# install and test ruby-vips to confirm we can pick up the libraries
# correctly
# we need ruby-dev to install ruby-ffi
ENV LD_LIBRARY_PATH=$PREFIX/lib
RUN apt-get install -y ruby-dev
RUN gem install ruby-vips
RUN ruby -e 'require "vips"; puts "ruby-vips: libvips #{Vips::LIBRARY_VERSION}"'
