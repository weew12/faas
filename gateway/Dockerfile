# 注释 不用做许可证检查
# FROM --platform=${BUILDPLATFORM:-linux/amd64} ghcr.io/openfaas/license-check:0.4.1 AS license-check

FROM --platform=${BUILDPLATFORM:-linux/amd64} golang:1.24 AS build

ENV GO111MODULE=on
ENV CGO_ENABLED=0

ARG TARGETPLATFORM
ARG BUILDPLATFORM
ARG TARGETOS
ARG TARGETARCH

ARG GIT_COMMIT
ARG VERSION

# 注释 不用做许可证检查
# COPY --from=license-check /license-check /usr/bin/

# WORKDIR /go/src/github.com/openfaas/faas/gateway
# 修改 workdir 
WORKDIR /gateway

COPY vendor         vendor
COPY go.mod         go.mod
COPY go.sum         go.sum

COPY handlers       handlers
COPY metrics        metrics
COPY requests       requests

COPY types          types
COPY plugin         plugin
COPY version        version
COPY scaling        scaling
COPY pkg            pkg
COPY main.go        .

# 注释 不用做许可证检查
# RUN license-check -path ./ --verbose=false "Alex Ellis" "OpenFaaS Authors" "OpenFaaS Author(s)"

# Run a gofmt and exclude all vendored code.
# 注释 跳过测试 加速构建过程
# RUN test -z "$(gofmt -l $(find . -type f -name '*.go' -not -path "./vendor/*"))"
# RUN go test $(go list ./... | grep -v integration | grep -v /vendor/ | grep -v /template/) -cover

RUN CGO_ENABLED=${CGO_ENABLED} GOOS=${TARGETOS} GOARCH=${TARGETARCH} go build --ldflags "-s -w \
    -X \"github.com/openfaas/faas/gateway/version.GitCommitSHA=${GIT_COMMIT}\" \
    -X \"github.com/openfaas/faas/gateway/version.Version=${VERSION}\" \
    -X github.com/openfaas/faas/gateway/types.Arch=${TARGETARCH}" \
    -o gateway .

FROM --platform=${TARGETPLATFORM:-linux/amd64} alpine:3.22.1 AS ship

# 注释 用于为 Docker 镜像添加元数据标签 不影响镜像使用
# LABEL org.label-schema.license="OpenFaaS CE EULA - non-commercial" \
#     org.label-schema.vcs-url="https://github.com/openfaas/faas" \
#     org.label-schema.vcs-type="Git" \
#     org.label-schema.name="openfaas/faas" \
#     org.label-schema.vendor="openfaas" \
#     org.label-schema.docker.schema-version="1.0"

# 注释 用于为 Docker 镜像添加元数据标签 不影响镜像使用
# LABEL org.opencontainers.image.source=https://github.com/openfaas/faas

# 更改 apk 仓库源 将 /etc/apk/repositories 里面 dl-cdn.alpinelinux.org 的 改成 mirrors.aliyun.com
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && apk update

RUN addgroup -S app \
    && adduser -S -g app app \
    && apk add --no-cache ca-certificates

WORKDIR /home/app

EXPOSE 8080
EXPOSE 8082
ENV http_proxy=""
ENV https_proxy=""

# COPY --from=build /go/src/github.com/openfaas/faas/gateway/gateway    .
# 复制构建 stage 的输出位置的二进制文件到当前位置
COPY --from=build /gateway    .
COPY assets     assets

ARG TARGETPLATFORM

RUN if [ "$TARGETPLATFORM" = "linux/arm/v7" ] ; then sed -ie s/x86_64/armhf/g assets/script/funcstore.js ; elif [ "$TARGETPLATFORM" = "linux/arm64" ] ; then sed -ie s/x86_64/arm64/g assets/script/funcstore.js; fi

RUN chown -R app:app ./

USER app

CMD ["./gateway"]

