ARG BUILDPLATFORM=linux/amd64

FROM --platform=$BUILDPLATFORM docker.io/curlimages/curl:7.75.0  as await
WORKDIR /tmp
ARG TARGETARCH=amd64
ARG LINKERD_AWAIT_VERSION=v0.2.1
RUN curl -fsSvLo linkerd-await https://github.com/olix0r/linkerd-await/releases/download/release%2F${LINKERD_AWAIT_VERSION}/linkerd-await-${LINKERD_AWAIT_VERSION}-${TARGETARCH} && chmod +x linkerd-await

# Precompile key slow-to-build dependencies
FROM --platform=$BUILDPLATFORM golang:1.14.15-alpine as go-deps
WORKDIR /linkerd-build
COPY go.mod go.sum ./
COPY bin/install-deps bin/
RUN go mod download
ARG TARGETARCH
RUN ./bin/install-deps $TARGETARCH

## compile controller service
FROM go-deps as golang
WORKDIR /linkerd-build
COPY controller/gen controller/gen
# TODO: remove when BuildResource is refactored
# https://github.com/linkerd/linkerd2/issues/5589
COPY viz/metrics-api/gen/viz viz/metrics-api/gen/viz
COPY pkg pkg
COPY controller controller
COPY charts/patch charts/patch
COPY charts/partials charts/partials
COPY multicluster multicluster

# Generate static templates
# TODO: `go generate` does not honor -mod=readonly
RUN go generate -mod=readonly ./pkg/charts/static
ARG TARGETARCH
RUN CGO_ENABLED=0 GOOS=linux GOARCH=$TARGETARCH go build -o /out/controller -tags prod -mod=readonly -ldflags "-s -w" ./controller/cmd

## package runtime
FROM scratch
COPY --from=await /tmp/linkerd-await /
COPY LICENSE /linkerd/LICENSE
COPY --from=golang /out/controller /controller
# for heartbeat (https://versioncheck.linkerd.io/version.json)
COPY --from=golang /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

ARG LINKERD_VERSION
ENV LINKERD_CONTAINER_VERSION_OVERRIDE=${LINKERD_VERSION}
ENTRYPOINT ["/linkerd-await", "--", "/controller"]
