FROM nginx:latest
COPY default.conf /etc/nginx/conf.d/default.conf
COPY ./app /app
RUN usermod -u 1001 nginx && groupmod -g 1001 nginx
RUN chown -R nginx:nginx /app && chmod -R 755 /app && \
        chown -R nginx:nginx /var/cache/nginx && \
        chown -R nginx:nginx /var/log/nginx && \
        chown -R nginx:nginx /etc/nginx/conf.d
RUN touch /var/run/nginx.pid && \
        chown -R nginx:nginx /var/run/nginx.pid
USER nginx
EXPOSE 8000

