# Grafana
Установка Grafana и Prometheus

---

В данной инструкции мы разберемся с вами правильно установить и настроить одну из самых популярных система мониторинга Grafana + Prometheus. Установку мы буде производить в Docker. В статье мы подробно рассмотрим все этапы установки, а в самом конце вы найдете готовый compose файл для быстрой установки.

Установка Docker
Вы можете установить докер в ручную либо заказать уже готовый виртуальный сервер с Docker и Portainer у нас на сайте

Установим необходимые пакеты

---

sudo apt update
sudo apt install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release


Добави GPG ключ Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
Добавим стабильный репозиторий Docker

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
Обновим пакеты и установим Docker

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
Для удобства управления контейнерами можно дополнительно установить Portainer. После установки вебинтерфейс Portainer будет доступен по адресу https://ip:9443

docker run -d -p 8000:8000 -p 9443:9443 --name portainer \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:latest
Установим docker-compose

apt  install docker-compose
На данном подготовку для установки Grafana + Portainer мы закончили.

Установка Grafana
Для установки последней стабильной вервии Grafana достаточно выполнить следующую команду

docker run -d -p 3000:3000 --name grafana grafana/grafana-enterprise

После завершения установки веб интерфейс будет доступен по адресу http://ip:3000. Вам будет необходимо ввести логин admin и пароль admin, после чего вам будет предложено указать новый пароль.

Установка Prometheus + Node Exporter
Установим Prometheus и для сбора статистики с хост системы(системы на которой установлен сервер Grafana) дополнительно установим Node Exporter.

Для упрощения установки мы создадим файл docker-compose.yml со следующим содержимым:

version: '3.3'

networks:
  monitoring:
    driver: bridge
    
volumes:
  prometheus_data: {}

services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - 9100:9100
    networks:
      - monitoring
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    ports:
      - 9090:9090
    networks:
      - monitoring
Для сбора статистики Node Exporter мы смонтировали /proc /sys и / в режиме чтения.

Для Prometheus нам необходимо указать папку в которой будет располагаться файл конфигурации prometheus.yml. В примере выше путь указан к корневой папке с которой будет запущен compose файл.

Создадим директорию prometheus в которой создадим файл prometheus.yml со следующим содержимым:

global:
  scrape_interval:     15s

scrape_configs:
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
    - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
    - targets: ["node-exporter:9100"]
Вернемся на уровень выше, где находится файл docker-compose.yml и выполним установку

docker-compose up -d

После установки проверим доступность Prometheus который будет доступен по адресу http://ip:9090, а метрики который собираются на хост системе будут доступы по адресу http://ip:9100.

Теперь остался последний шаг — добавить в Grafana в качестве источника получения данных Prometheus.


Переходим в Grafana — Configuration — Data sources и нажимаем Add data source и выбираем Prometheus.


В поле URL вводим адрес и порт по которому доступен Prometheus в нашем случаи это http://ip:9090


Прокручиваем в низ страницы и нажимаем Save& test и после успешной проверки Prometheus будет добавлен в Grafana.

Последний шаг установки — установка дашборда Node Exporter для сбора статистики с хост системы.

Переходим в Dashboards (четыре квадрата в боковом меню) — Brows — и нажимаем Import.


В поле импорта указываем ID 1860 и нажимаем на кнопку Load


Нам будет предложено сменить название дашборда и папку в которой он будет располагаться и в поле Prometheus необходимо выбрать название источника данных которое мы добавили.


После того как вы нажмете кнопку Import вы попадает в установленный дашборд в котором уже будут отображаться собранные данные.

Для того чтобы установить установить всю связку необходимо создать файл docker-compose.yml в который необходимо добавить следующее

version: '3.3'

networks:
  monitoring:
    driver: bridge
    
volumes:
  prometheus_data: {}

services:
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - 9100:9100
    networks:
      - monitoring
  grafana:
    image: grafana/grafana-enterprise
    container_name: grafana
    restart: unless-stopped
    ports:
      - 3000:3000
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    ports:
      - 9090:9090
    networks:
      - monitoring
И выполнить команду:

docker-compose up -d
