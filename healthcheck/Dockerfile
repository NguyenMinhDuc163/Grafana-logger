ARG BASE_IMAGE=busybox:1.37.0-musl
FROM busybox:1.37.0-musl AS healthcheck-tools

FROM ${BASE_IMAGE}
COPY --from=healthcheck-tools /bin/busybox /busybox
