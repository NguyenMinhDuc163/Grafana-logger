ARG GRAFANA_IMAGE=grafana/grafana:13.1.0
FROM ${GRAFANA_IMAGE}

ARG VICTORIALOGS_PLUGIN_VERSION=0.27.1

USER root
RUN grafana cli --pluginsDir "${GF_PATHS_PLUGINS}" plugins install \
      victoriametrics-logs-datasource "${VICTORIALOGS_PLUGIN_VERSION}" \
    && chown -R grafana:root "${GF_PATHS_PLUGINS}"
USER grafana
