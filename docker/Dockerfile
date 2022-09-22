# FROM ubuntu:latest
FROM nginx:stable-alpine

# install nginx and tzdata.
# RUN apt-get update && \
#   apt-get install -y tzdata \
#   nginx

# set timezone
# ENV TZ Asia/Tokyo

# add nginx and config files.
COPY docker/conf/nginx.conf /etc/nginx/nginx.conf
COPY docker/conf/default.conf /etc/nginx/conf.d/default.conf
COPY build/ /var/www/html/

# start nginx process.
# EXPOSE 80
# ENTRYPOINT ["nginx", "-g", "daemon off;"]