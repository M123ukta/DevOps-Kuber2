FROM nginx:1.27-alpine

# Application version displayed in the page footer.
ARG APP_VERSION=v1.0

# Set FAULTY=true to remove the health endpoint.
ARG FAULTY=false

# Copy application files.
COPY index.html style.css health.html /usr/share/nginx/html/

# Stamp the version and optionally remove the health endpoint.
RUN sed -i "s/__VERSION__/${APP_VERSION}/g" /usr/share/nginx/html/index.html && if [ "${FAULTY}" = "true" ]; then rm -f /usr/share/nginx/html/health.html; fi

EXPOSE 80

# Check whether the health endpoint responds successfully.
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=2 CMD wget -qO- http://127.0.0.1/health.html || exit 1

# Start Nginx in the foreground.
CMD ["nginx", "-g", "daemon off;"]