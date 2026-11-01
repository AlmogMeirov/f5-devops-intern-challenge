# Minimal reverse-proxy image.
#
# Was: ubuntu:24.04 + apt-get install nginx curl htop vim git, running as root.
# Now: the official nginx image on Alpine, "slim" variant (nginx only - no njs,
# perl, or extra modules), pinned to the 1.30 stable branch so rebuilds pick up
# patch releases but never a surprise major/minor jump.
FROM nginx:1.30-alpine-slim

# Our nginx.conf replaces the stock one completely and doesn't include
# conf.d/, so remove the unused default site rather than ship dead config.
RUN rm -f /etc/nginx/conf.d/default.conf

# Owned by root and not writable by the runtime user: a compromised nginx
# worker can't rewrite its own config.
COPY nginx.conf /etc/nginx/nginx.conf

# Drop root. uid/gid 101 is the "nginx" user that ships in the official image;
# numeric so orchestrators (e.g. Kubernetes runAsNonRoot) can verify it.
USER 101:101

EXPOSE 8080

CMD ["nginx", "-g", "daemon off;"]
