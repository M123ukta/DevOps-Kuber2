FROM nginx:1.27-alpine

# Application version displayed in the page footer.
ARG APP_VERSION=v1.0

# Set to true to build a deliberately faulty release.
ARG FAULTY=false

# Make the build arguments available as image environment variables if needed.
ENV APP_VERSION=${APP_VERSION}
ENV FAULTY=${FAULTY}

# Copy the application files into the Nginx web root.
COPY index.html style.css health.html /usr/share/nginx/html/

# Stamp the version into the page and introduce the controlled fault.
RUN sed -i "s/__VERSION__/${APP_VERSION}/g" /usr/share/nginx/html/index.html 
    && if [ "${FAULTY}" = "true" ]; then 
         rm -f /usr/share/nginx/html/health.html; 
       fi

# Nginx listens on port 80.
EXPOSE 80

# Mark the container unhealthy when the health endpoint is unavailable.
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=2 
    CMD wget -qO- http://127.0.0.1/health.html || exit 1

# Start Nginx in the foreground.
CMD ["nginx", "-g", "daemon off;"]