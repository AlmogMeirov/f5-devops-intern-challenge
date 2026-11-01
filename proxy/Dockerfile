# Proxy image inherited from the previous team.

FROM ubuntu:24.04

RUN apt-get update && \
    apt-get install -y nginx curl htop vim git

# Copy our config only AFTER nginx is installed. If the file already exists
# when the nginx-common package is configured, dpkg treats it as a locally
# modified conffile and stops to ask what to do - and a docker build has no
# stdin to answer with ("end of file on stdin at conffile prompt").
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
