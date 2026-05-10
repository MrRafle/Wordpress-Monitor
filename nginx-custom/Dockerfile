FROM nginx:latest

RUN apt-get update && apt-get install -y \
    supervisor curl wget ca-certificates \
    && wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_7.0-1+debian12_all.deb \
    && dpkg -i zabbix-release_7.0-1+debian12_all.deb \
    && apt-get update && apt-get install -y zabbix-agent2 \
    && rm -rf /var/lib/apt/lists/* zabbix-release*.deb

COPY supervisord.conf /etc/supervisord.conf
COPY zabbix_agent2.conf /etc/zabbix/zabbix_agent2.conf

RUN echo "<h1>✅ Nginx + Zabbix Agent2 успешно работает!</h1>" > /usr/share/nginx/html/index.html && \
    mkdir -p /var/log/zabbix /var/run/zabbix && \
    chown -R root:root /var/log/zabbix /var/run/zabbix

EXPOSE 80
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
