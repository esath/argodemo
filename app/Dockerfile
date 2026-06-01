FROM nginx:1.27-alpine

# Copy the web app files into the nginx html directory
COPY index.html /usr/share/nginx/html/index.html

# Expose port 8080 (OpenShift runs containers as non-root,
# so we use port 8080 instead of the default 80)
EXPOSE 8080

# Make the image compatible with OpenShift arbitrary UIDs.
RUN sed -i 's/listen\s*80;/listen 8080;/g' /etc/nginx/conf.d/default.conf && \
    sed -i '/^user\s\+nginx;/d' /etc/nginx/nginx.conf && \
    sed -i 's|^pid\s\+/var/run/nginx.pid;|pid /tmp/nginx.pid;|' /etc/nginx/nginx.conf && \
    rm -f /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh && \
    mkdir -p /tmp/client_temp /tmp/proxy_temp /tmp/fastcgi_temp /tmp/uwsgi_temp /tmp/scgi_temp && \
    chgrp -R 0 /var/cache/nginx /var/log/nginx /etc/nginx/conf.d /usr/share/nginx/html /tmp && \
    chmod -R g=u /var/cache/nginx /var/log/nginx /etc/nginx/conf.d /usr/share/nginx/html /tmp

CMD ["nginx", "-g", "daemon off;"]
