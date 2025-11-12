FROM nginxinc/nginx-unprivileged:alpine
COPY . /usr/share/nginx/html
COPY docker/nginx.conf /etc/nginx/nginx.conf
HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD wget -q --spider http://localhost:8080/index.html && test -f /usr/share/nginx/html/fish_doodle_classifier.onnx || exit 1
EXPOSE 8080
