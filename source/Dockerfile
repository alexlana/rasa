# The default Docker image
ARG IMAGE_BASE_NAME
ARG BASE_IMAGE_HASH
ARG BASE_BUILDER_IMAGE_HASH

FROM ${IMAGE_BASE_NAME}:base-builder-${BASE_BUILDER_IMAGE_HASH} as builder
# copy files
COPY . /build/

# change working directory
WORKDIR /build

##################################################################
## ARM64 - INI
RUN pip3 install --upgrade pip
RUN python3 -m pip install --upgrade setuptools
RUN pip3 install --no-cache-dir  --force-reinstall -Iv grpcio
RUN apt-get update -qq && apt-get install -y libhdf5-dev pkg-config wget git
RUN pip install h5py
RUN apt-get install -y wget

# BAZEL
RUN wget https://github.com/bazelbuild/bazel/releases/download/3.7.2/bazel-3.7.2-linux-arm64
RUN chmod +x bazel-3.7.2-linux-arm64
RUN mv bazel-3.7.2-linux-arm64 /usr/local/bin/bazel
RUN bazel version
RUN which bazel

# # TF FROM KUMATEA
# ADD ["https://github.com/KumaTea/tensorflow-aarch64/releases/download/v2.6/tensorflow-2.6.0-cp38-cp38-linux_aarch64.whl", "/tensorflow-2.6.0-cp38-cp38-linux_aarch64.whl"]
# RUN python3 -m pip install --no-cache-dir /tensorflow-2.6.0-cp38-cp38-linux_aarch64.whl
# RUN rm /tensorflow-2.6.0-cp38-cp38-linux_aarch64.whl
# COPY ./_repo/bashrc /etc/bash.bashrc
# RUN chmod a+rwx /etc/bash.bashrc

# TF - LOCAL BUILD
RUN pip install -U pip numpy wheel
RUN pip install -U keras_preprocessing --no-deps
RUN git clone https://github.com/tensorflow/tensorflow.git
# COPY ./_repo/tensorflow-2.6.0 /build/tensorflow
WORKDIR /build/tensorflow
RUN git checkout r2.6
RUN ./configure
RUN bazel build --local_ram_resources=4096 --local_cpu_resources=4 --verbose_failures //tensorflow/tools/pip_package:build_pip_package
WORKDIR /build

# # TF ADDONS FROM QENGINEERING
# RUN wget https://github.com/Qengineering/TensorFlow-Addons-Raspberry-Pi_64-bit/raw/main/tensorflow_addons-0.14.0.dev0-cp38-cp38-linux_aarch64.whl
# RUN python3 -m pip install --no-cache-dir tensorflow_addons-0.14.0.dev0-cp38-cp38-linux_aarch64.whl
# RUN rm tensorflow_addons-0.14.0.dev0-cp38-cp38-linux_aarch64.whl

# TF ADDONS - LOCAL BUILD
RUN git clone https://github.com/tensorflow/addons.git
WORKDIR /builds/addons
RUN python3 ./configure.py
RUN bazel build build_pip_pkg
RUN bazel-bin/build_pip_pkg artifacts
RUN pip install artifacts/tensorflow_addons-*.whl

# TF TEXT - LOCAL BUILD
RUN git clone https://github.com/tensorflow/text.git
WORKDIR /build/text
RUN ./oss_scripts/run_build.sh
RUN pip install $(find . -type f -name "*.whl")

# RUN apt-get install -y git
# RUN pip install --upgrade numpy
# RUN pip install --upgrade pip setuptools wheel
## ARM64 - FIM
##################################################################

# install dependencies
RUN python -m venv /opt/venv && \
  . /opt/venv/bin/activate && \
  pip install --no-cache-dir -U "pip==21.*"
RUN poetry install --no-dev --no-root --no-interaction && \
  poetry build -f wheel -n && \
  pip install --no-deps dist/*.whl && \
  rm -rf dist *.egg-info

# start a new build stage
FROM ${IMAGE_BASE_NAME}:base-${BASE_IMAGE_HASH} as runner

# copy everything from /opt
COPY --from=builder /opt/venv /opt/venv

# make sure we use the virtualenv
ENV PATH="/opt/venv/bin:$PATH"

# set HOME environment variable
ENV HOME=/app

# update permissions & change user to not run as root
WORKDIR /app
RUN chgrp -R 0 /app && chmod -R g=u /app && chmod o+wr /app
USER 1001

# create a volume for temporary data
VOLUME /tmp

# change shell
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# the entry point
EXPOSE 5005
ENTRYPOINT ["rasa"]
CMD ["--help"]
