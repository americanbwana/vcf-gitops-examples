# Non-root web server that satisfies the Kubernetes "restricted" Pod Security level.
# Listens on 8080 (unprivileged port) and runs as uid 101 (nginx).
# Build for the platform VKS nodes use:
#   docker buildx build --platform linux/amd64 -t vcf-gitops-guestbook:dev --load .
FROM nginxinc/nginx-unprivileged:1.27-alpine

COPY nginx/default.conf /etc/nginx/conf.d/default.conf
COPY html/ /usr/share/nginx/html/

# The base image already runs as the non-root "nginx" user (uid 101).
# Declare it numerically so runAsNonRoot can be verified by the kubelet.
USER 101

EXPOSE 8080
