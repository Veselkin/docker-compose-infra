ROM debian:stable-slim

# Устанавливаем icecast2 и gettext для замены переменных
RUN apt-get update && \
    apt-get install -y icecast2 gettext-base && \
    rm -rf /var/lib/apt/lists/*

RUN mkdir -p /var/log/icecast && \
    chown -R 101:101 /var/log/icecast

# Копируем шаблон
COPY icecast.xml.template /etc/icecast2/icecast.xml.template

# Скрипт запуска: подставляем переменные и запускаем icecast
CMD envsubst < /etc/icecast2/icecast.xml.template > /etc/icecast2/icecast.xml && \
    chown 101:101 /etc/icecast2/icecast.xml && \
    exec icecast2 -c /etc/icecast2/icecast.xml