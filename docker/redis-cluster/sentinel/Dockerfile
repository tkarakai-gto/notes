FROM redis:4.0-alpine

#
ENV SENTINEL_QUORUM 2
ENV SENTINEL_DOWN_AFTER 10000
ENV SENTINEL_FAILOVER 10000

EXPOSE 26379

RUN mkdir -p /etc/redis
COPY sentinel.conf /etc/redis/sentinel.conf
COPY sentinel-entrypoint.sh /usr/local/bin/

RUN chown redis:redis /etc/redis/* && \
    chmod +x /usr/local/bin/sentinel-entrypoint.sh

ENTRYPOINT ["sentinel-entrypoint.sh"]
