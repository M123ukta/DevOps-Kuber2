FROM nginx:1.27-alpine

# APP_VERSION is stamped into the page footer (versioned releases).
# FAULTY=true builds a deliberately broken release: the /health.html endpoint is
# removed, so the Kubernetes readiness probe fails and the rollout never completes.
ARG APP_VERSION=v1.0
ARG FAULTY=false

COPY index.html style.css health.html /usr/share/nginx/html/

RUN sed -i "s/__VERSION__/${APP_VERSION}/g" /usr/share/nginx/html/index.html \
 && if [ "${FAULTY}" = "true" ]; then rm -f /usr/share/nginx/html/health.html; fi

EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost/health.html || exit 1
