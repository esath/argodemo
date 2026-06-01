FROM nginx:1.27-alpine

# Copy the web app files into the nginx html directory
COPY index.html /usr/share/nginx/html/index.html

# Expose port 8080 (OpenShift runs containers as non-root,
# so we use port 8080 instead of the default 80)
EXPOSE 8080

# Use a custom nginx config that listens on 8080
RUN sed -i 's/listen\s*80;/listen 8080;/g' /etc/nginx/conf.d/default.conf && \
    # Allow nginx to run as non-root user
    chown -R nginx:nginx /var/cache/nginx /var/log/nginx /etc/nginx/conf.d && \
    chmod -R g+w /var/cache/nginx /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown nginx:nginx /var/run/nginx.pid

USER nginx

CMD ["nginx", "-g", "daemon off;"]
