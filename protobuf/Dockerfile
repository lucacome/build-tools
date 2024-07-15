# gRPC core version that applies to C++, C#, Objective-C, PhP, Python, Ruby
ARG GRPC_VERSION=1.65.0
ARG PROTOBUF_C_VERSION=1.5.0
ARG GRPC_WEB_VERSION=1.5.0
ARG GRPC_JAVA_VERSION=1.65.1

ARG PROTOC_GEN_GO_VERSION=1.32.0
ARG PROTOC_GEN_GO_GRPC_VERSION=1.52.3

# use the lastest commit from master
ARG PROTOC_GEN_GOGO_VERSION=f67b8970b736e53dbd7d0a27146c8f1ac52f74e5
ARG PROTOC_GEN_LINT_VERSION=0.3.0
ARG GRPC_GATEWAY_VERSION=2.19.1
ARG PROTOC_GEN_PARQUET_VERSION=0.4.3
ARG UPX_VERSION=4.2.2

FROM --platform=$BUILDPLATFORM tonistiigi/xx:1.4.0 AS xx

FROM --platform=$BUILDPLATFORM alpine:3.20 AS protoc_builder_base
RUN apk add --no-cache \
	alpine-sdk \
	automake \
autoconf \
libtool \
	clang \
	cmake \
	lld \
	gcompat \
	protobuf-dev \
    zlib-dev re2-dev c-ares-dev

COPY --from=xx / /
ARG TARGETPLATFORM

RUN xx-apk add --no-cache xx-cxx-essentials zlib-dev zlib linux-headers openssl-dev re2-dev c-ares-dev

FROM protoc_builder_base AS protoc_builder

RUN mkdir -p /out

ARG GRPC_VERSION
ADD https://github.com/grpc/grpc.git#v${GRPC_VERSION} /grpc
RUN xx-info env && \
    ln -s /grpc/third_party/protobuf /protobuf && \
    mkdir -p /grpc/cmake/build && \
    cd /grpc/cmake/build && \
    cmake $(xx-clang --print-cmake-defines) \
    # -GNinja \
    -DBUILD_SHARED_LIBS=ON \
    -DCMAKE_INSTALL_PREFIX=/out \
    -DCMAKE_BUILD_TYPE=Release \
    -DgRPC_INSTALL=ON \
    -DgRPC_ABSL_PROVIDER=package     \
    -DgRPC_PROTOBUF_PROVIDER=module \
    -DgRPC_RE2_PROVIDER=package      \
    -DgRPC_SSL_PROVIDER=package      \
    -DgRPC_ZLIB_PROVIDER=package \
    -DgRPC_BUILD_TESTS=OFF \
    ../.. && \
    make -j$(nproc) && \
	make install && \
	xx-verify /out/lib/libgrpc.so

FROM protoc_builder AS protobuf_builder

ARG PROTOBUF_C_VERSION
ADD https://github.com/protobuf-c/protobuf-c/releases/download/protobuf-c-${PROTOBUF_C_VERSION}.tar.gz /protobuf-c.tar.gz

RUN mkdir -p /protobuf-c && \
	tar xzf /protobuf-c.tar.gz --strip 1 -C /protobuf-c && \
    cd /protobuf-c && \
    # export LD_LIBRARY_PATH=/usr/lib:/usr/lib64 && \
    # export PKG_CONFIG_PATH=/usr/lib64/pkgconfig && \
    ./autogen.sh && \
    ./configure --host=$(xx-clang --print-target-triple) && \
    make && make install DESTDIR=/out

ARG GRPC_JAVA_VERSION
ADD https://github.com/grpc/grpc-java.git#v${GRPC_JAVA_VERSION} /grpc-java

RUN cd /grpc-java && \
    g++ \
    -I. -I/usr/include \
    compiler/src/java_plugin/cpp/*.cpp \
    -L/usr/lib64 \
    -lprotoc -lprotobuf -lpthread -s \
    -o protoc-gen-grpc-java && \
    install -Ds protoc-gen-grpc-java /out/usr/bin/protoc-gen-grpc-java && \
    rm -Rf /grpc-java && \
    rm -Rf /grpc

ARG GRPC_WEB_VERSION
RUN mkdir -p /grpc-web && \
    curl -sSL https://api.github.com/repos/grpc/grpc-web/tarball/${GRPC_WEB_VERSION} | tar xz --strip 1 -C /grpc-web && \
    cd /grpc-web && \
    make install-plugin && \
    install -Ds /usr/local/bin/protoc-gen-grpc-web /out/usr/bin/protoc-gen-grpc-web


############# Build go ################
FROM golang:1.22-alpine as go_builder
RUN apk add --no-cache build-base curl git

ARG PROTOC_GEN_GO_GRPC_VERSION
RUN mkdir -p ${GOPATH}/src/github.com/grpc/grpc-go && \
    curl -sSL https://api.github.com/repos/grpc/grpc-go/tarball/v${PROTOC_GEN_GO_GRPC_VERSION} | tar xz --strip 1 -C ${GOPATH}/src/github.com/grpc/grpc-go &&\
    cd ${GOPATH}/src/github.com/grpc/grpc-go/cmd/protoc-gen-go-grpc && \
    go build -ldflags '-w -s' -o /golang-protobuf-out/protoc-gen-go-grpc . && \
    install -Ds /golang-protobuf-out/protoc-gen-go-grpc /out/usr/bin/protoc-gen-go-grpc

ARG PROTOC_GEN_GO_VERSION
RUN mkdir -p ${GOPATH}/src/github.com/protocolbuffers/protobuf-go && \
    curl -sSL https://api.github.com/repos/protocolbuffers/protobuf-go/tarball/v${PROTOC_GEN_GO_VERSION} | tar xz --strip 1 -C ${GOPATH}/src/github.com/protocolbuffers/protobuf-go &&\
    cd ${GOPATH}/src/github.com/protocolbuffers/protobuf-go && \
    go build -ldflags '-w -s' -o /golang-protobuf-out/protoc-gen-go ./cmd/protoc-gen-go && \
    install -Ds /golang-protobuf-out/protoc-gen-go /out/usr/bin/protoc-gen-go

ARG PROTOC_GEN_GOGO_VERSION
RUN mkdir -p ${GOPATH}/src/github.com/gogo/protobuf && \
    curl -sSL https://api.github.com/repos/gogo/protobuf/tarball/${PROTOC_GEN_GOGO_VERSION} | tar xz --strip 1 -C ${GOPATH}/src/github.com/gogo/protobuf &&\
    cd ${GOPATH}/src/github.com/gogo/protobuf && \
    go build -ldflags '-w -s' -o /gogo-protobuf-out/protoc-gen-gofast ./protoc-gen-gofast && \
    install -Ds /gogo-protobuf-out/protoc-gen-gofast /out/usr/bin/protoc-gen-gofast && \
    go build -ldflags '-w -s' -o /gogo-protobuf-out/protoc-gen-gogo ./protoc-gen-gogo && \
    install -Ds /gogo-protobuf-out/protoc-gen-gogo /out/usr/bin/protoc-gen-gogo && \
    go build -ldflags '-w -s' -o /gogo-protobuf-out/protoc-gen-gogofast ./protoc-gen-gogofast && \
    install -Ds /gogo-protobuf-out/protoc-gen-gogofast /out/usr/bin/protoc-gen-gogofast && \
    go build -ldflags '-w -s' -o /gogo-protobuf-out/protoc-gen-gogofaster ./protoc-gen-gogofaster && \
    install -Ds /gogo-protobuf-out/protoc-gen-gogofaster /out/usr/bin/protoc-gen-gogofaster && \
    mkdir -p /out/usr/include/github.com/gogo/protobuf/protobuf/google/protobuf && \
    install -D $(find ./protobuf/google/protobuf -name '*.proto') -t /out/usr/include/github.com/gogo/protobuf/protobuf/google/protobuf && \
    install -D ./gogoproto/gogo.proto /out/usr/include/github.com/gogo/protobuf/gogoproto/gogo.proto

ARG PROTOC_GEN_LINT_VERSION
ARG TARGETARCH
RUN cd / && \
    curl -sSLO https://github.com/ckaznocha/protoc-gen-lint/releases/download/v${PROTOC_GEN_LINT_VERSION}/protoc-gen-lint_linux_${TARGETARCH}.zip && \
    mkdir -p /protoc-gen-lint-out && \
    cd /protoc-gen-lint-out && \
    unzip -q /protoc-gen-lint_linux_${TARGETARCH}.zip && \
    install -D /protoc-gen-lint-out/protoc-gen-lint /out/usr/bin/protoc-gen-lint

ARG GRPC_GATEWAY_VERSION
RUN mkdir -p ${GOPATH}/src/github.com/grpc-ecosystem/grpc-gateway && \
    curl -sSL https://api.github.com/repos/grpc-ecosystem/grpc-gateway/tarball/v${GRPC_GATEWAY_VERSION} | tar xz --strip 1 -C ${GOPATH}/src/github.com/grpc-ecosystem/grpc-gateway && \
    cd ${GOPATH}/src/github.com/grpc-ecosystem/grpc-gateway && \
    go build -ldflags '-w -s' -o /grpc-gateway-out/protoc-gen-grpc-gateway ./protoc-gen-grpc-gateway && \
    go build -ldflags '-w -s' -o /grpc-gateway-out/protoc-gen-openapiv2 ./protoc-gen-openapiv2 && \
    install -Ds /grpc-gateway-out/protoc-gen-grpc-gateway /out/usr/bin/protoc-gen-grpc-gateway && \
    install -Ds /grpc-gateway-out/protoc-gen-openapiv2 /out/usr/bin/protoc-gen-openapiv2 && \
    mkdir -p /out/usr/include/protoc-gen-openapiv2/options && \
    install -D $(find ./protoc-gen-openapiv2/options -name '*.proto') -t /out/usr/include/protoc-gen-openapiv2/options

ARG PROTOC_GEN_PARQUET_VERSION
RUN mkdir -p ${GOPATH}/src/github.com/atoulme/protoc-gen-parquet && \
    curl -sSL https://api.github.com/repos/atoulme/protoc-gen-parquet/tarball/v${PROTOC_GEN_PARQUET_VERSION} | tar xz --strip 1 -C ${GOPATH}/src/github.com/atoulme/protoc-gen-parquet && \
    cd ${GOPATH}/src/github.com/atoulme/protoc-gen-parquet && \
    go build -ldflags '-w -s' -o bin/protoc-gen-parquet . && \
    install -Ds bin/protoc-gen-parquet /out/usr/bin/protoc-gen-parquet && \
    mkdir -p /out/usr/include/protoc-gen-parquet/options && \
    install -D $(find ./parquet_options -name '*.proto') -t /out/usr/include/protoc-gen-parquet/options


############# Packer ###############
FROM alpine:3.18 as packer
RUN apk add --no-cache curl

ARG UPX_VERSION
ARG TARGETARCH
RUN mkdir -p /upx && curl -sSL https://github.com/upx/upx/releases/download/v${UPX_VERSION}/upx-${UPX_VERSION}-${TARGETARCH}_linux.tar.xz | tar xJ --strip 1 -C /upx && \
    install -D /upx/upx /usr/local/bin/upx

# Use all output including headers and protoc from protoc_builder
COPY --from=protoc_builder /out/ /out/

# Integrate all output from go_builder
COPY --from=go_builder /out/ /out/

RUN upx --lzma $(find /out/usr/bin/ \
    -type f -name 'grpc_*' \
    -not -name 'grpc_csharp_plugin' \
    -not -name 'grpc_node_plugin' \
    -not -name 'grpc_php_plugin' \
    -not -name 'grpc_ruby_plugin' \
    -not -name 'grpc_python_plugin' \
    -or -name 'protoc-gen-*' \
    )
RUN find /out -name "*.a" -delete -or -name "*.la" -delete


############# Final Image ###############
FROM alpine:3.18
LABEL maintainer="The OpenTelemetry Authors"
COPY --from=packer /out/ /
RUN apk add --no-cache bash libstdc++ && \
    ln -s /usr/bin/grpc_cpp_plugin /usr/bin/protoc-gen-grpc-cpp && \
    ln -s /usr/bin/grpc_csharp_plugin /usr/bin/protoc-gen-grpc-csharp && \
    ln -s /usr/bin/grpc_objective_c_plugin /usr/bin/protoc-gen-grpc-objc && \
    ln -s /usr/bin/grpc_node_plugin /usr/bin/protoc-gen-grpc-js && \
    ln -s /usr/bin/grpc_php_plugin /usr/bin/protoc-gen-grpc-php && \
    ln -s /usr/bin/grpc_python_plugin /usr/bin/protoc-gen-grpc-python && \
    ln -s /usr/bin/grpc_ruby_plugin /usr/bin/protoc-gen-grpc-ruby
COPY protoc-wrapper /usr/bin/protoc-wrapper
ENV LD_LIBRARY_PATH='/usr/lib:/usr/lib64:/usr/lib/local'
ENTRYPOINT ["protoc-wrapper", "-I/usr/include"]
