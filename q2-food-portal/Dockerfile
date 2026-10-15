FROM nginx:1.27-alpine

# APP_VERSION is stamped into the page (index.html must contain __VERSION__).
# FAULTY=true builds a deliberately broken release: /health.html is removed, so the
# Kubernetes readiness probe fails and the new pods never become Ready.
ARG APP_VERSION=v1
ARG FAULTY=false

COPY index.html style.css health.html /usr/share/nginx/html/

RUN sed -i "s/__VERSION__/${APP_VERSION}/g" /usr/share/nginx/html/index.html \
 && if [ "${FAULTY}" = "true" ]; then rm -f /usr/share/nginx/html/health.html; fi

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost/index.html || exit 1