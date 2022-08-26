FROM registry.access.redhat.com/ubi9/nginx-120

# Configure NGINX.
ADD compliance-view/nginx.conf "${NGINX_CONF_PATH}"
ADD compliance-view/nginx-default-cfg/*.conf "${NGINX_DEFAULT_CONF_PATH}"
ADD compliance-view/nginx-cfg/*.conf "${NGINX_CONFIGURATION_PATH}"
# Temporary index file that will be overwritten by PVC contents.
ADD compliance-view/index.html .

# Run script uses standard ways to run the application
CMD nginx -g "daemon off;"
