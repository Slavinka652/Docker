# Docker Kuvshinov

## Быстрый старт

* Для docker compose
  
```
git clone https://github.com/skl256/grafana_stack_for_docker.git && \   • Можно скачать git прямо из командной строки прописав Y
cd grafana_stack_for_docker && \
sudo mkdir -p /mnt/common_volume/swarm/grafana/config && \
sudo mkdir -p /mnt/common_volume/grafana/{grafana-config,grafana-data,prometheus-data,loki-data,promtail-data} && \
sudo chown -R $(id -u):$(id -g) {/mnt/common_volume/swarm/grafana/config,/mnt/common_volume/grafana} && \
touch /mnt/common_volume/grafana/grafana-config/grafana.ini && \
cp config/* /mnt/common_volume/swarm/grafana/config/ && \
mv grafana.yaml docker-compose.yaml && \

```

 * Устанавливаем последнею версию и утилитю docker-compose

```
sudo yum install curl && \
COMVER=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep 'tag_name' | cut -d" -f4) && \
sudo curl -L "https://github.com/docker/compose/releases/download/$COMVER/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose && \
sudo chmod +x /usr/bin/docker-compose && \
docker-compose --version && \
```

* Установка и настройка Docker

```
sudo yum install wget
sudo wget -P /etc/yum.repos.d/ https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce
sudo systemctl enable docker --now
docker compose up -d
```
              
## Конфигурация отдельных сервисов
### Prometheus
* После Устанвкой докера пишем ` sudo vi docker-compose.yaml ` (Обязательно что бы был расширения.yaml)Нас перекидывает на текстовый редактор:
  * Нас перекинет в текстовый редактор
   * Что-бы что-то изменить в тесковом редакторе нажмите `insert` на клавиатуре
   * Что бы сохранить что-то в этом документе нажимаем Esc пишем  `:wq!`
 В этом текставом редакторе мы должны поставить node-exporter после services

```
  node-exporter:
    image: prom/node-exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    container_name: exporter < запомнить название name
    hostname: exporter
    command:
      - --path.procfs=/host/proc
      - --path.sysfs=/host/sys
      - --collector.filesystem.ignored-mount-points
      - ^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)
    ports:
      - 9100:9100
    restart: unless-stopped
    environment:
      TZ: "Europe/Moscow"
    networks:
      - default
```
Далее пишем 
```
cd
cd /mnt/common_volume/swarm/grafana/config 
sudo vi prometheus.yaml 
```
* В этом файле нужно изменить первый ip Адрес на тот, который мы должны были запомнить targets: на exporter:9100,
  * Вставляем в первый targets 
  * После двоеточие цифры оставляем
 
  * ![image](https://github.com/user-attachments/assets/65dca558-0d9b-4881-9470-8f55a7c92f9b)


### Grafana
* переходим на сайт `localhost:3000`
  
* User & Password GRAFANA: admin
  * Код графаны: 3000
  * Код прометеуса: http://prometheus:9090

* в меню выбираем вкладку Dashboards и создаем Dashboard
  * ждем кнопку +Add visualization, а после "Configure a new data source"
  * выбираем Prometheus

* Connection
  * `http://prometheus:9090`

* Authentication
  * Basic authentication
    * User: `admin`
    * Password: `admin`
   * Нажимаем на Save & test и должно показывать зелёную галочку

* в меню выбираем вкладку Dashboards и создаем Dashboard
  * ждем кнопку "Import dashboard"
  * `Find and import dashboards for common applications at grafana.com/dashboards: 1860`  //ждем кнопку Load
  * Select Prometheus ждем кнопку "Import"
  
> [!NOTE]
> Всё должно выглядить как на скриншоте `(https://github.com/Slavinka652/Docker/blob/main/grafana_stack_for_docker/Screenshot_Prometheus.png)`

### VicroriaMetrics

* Для начала изменим docker-compose.yaml:
```
cd grafana_stack_for_docker
sudo vi docker-compose.yaml
```
После чего в в самом текстовом редакторе после `prometheus` вставляем
```
  vmagent:
    container_name: vmagent
    image: victoriametrics/vmagent:v1.105.0
    depends_on:
      - "victoriametrics"
    ports:
      - 8429:8429
    volumes:
      - vmagentdata:/vmagentdata
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - "--promscrape.config=/etc/prometheus/prometheus.yml"
      - "--remoteWrite.url=http://victoriametrics:8428/api/v1/write"
    restart: always
  # VictoriaMetrics instance, a single process responsible for
  # storing metrics and serve read requests.
  victoriametrics:
    container_name: victoriametrics
    image: victoriametrics/victoria-metrics:v1.105.0
    ports:
      - 8428:8428
      - 8089:8089
      - 8089:8089/udp
      - 2003:2003
      - 2003:2003/udp
      - 4242:4242
    volumes:
      - vmdata:/storage
    command:
      - "--storageDataPath=/storage"
      - "--graphiteListenAddr=:2003"
      - "--opentsdbListenAddr=:4242"
      - "--httpListenAddr=:8428"
      - "--influxListenAddr=:8089"
      - "--vmalert.proxyURL=http://vmalert:8880"
    restart: always
```
Сохраняем и выходим

* Захом в connection
  * там где мы писали `http://prometheus:9090` пишем  `http:victoriametrics:9090` И заменяем имя из "Prometheus-2" в "Vika"
  * нажимаем на dashboards add visualition выбираем "Vika"
  * снизу меняем на "code"
  * Переходим в терминал и пишем

```
echo -e "# TYPE OILCOINT_metric1 gauge\nOILCOINT_metric1 0" | curl --data-binary @- http://localhost:8428/api/v1/import/prometheus
curl -G 'http://localhost:8428/api/v1/query' --data-urlencode 'query=OILCOINT_metric1'

```
(Значение 0 меняем на любое другое)
* Копируем переменную OILCOINT_metric1 и вставляем в cod
* Нажимаем run 
![image](https://github.com/user-attachments/assets/49d9f76a-a545-485d-b191-ae67ebf6ddc3)
* Должно получится вот так

* Так же приложены фотки Терминала , если вдруг возникнут проблемы, так же приложен скрин с самой виктории метрикс!!!
![photo_2024-10-26_13-09-33](https://github.com/user-attachments/assets/114124e2-629f-474c-b880-0ebd3233522e)

![image](https://github.com/user-attachments/assets/9698ee24-f15b-4bcb-8821-4294623b5d65)



Sudo docker compose up -d 
![Screenshot_1](https://github.com/user-attachments/assets/5810ab47-e6bc-4d58-8750-f697ee9e1e8a)

![Screenshot_1](https://github.com/user-attachments/assets/74cf58ac-2a8a-4684-ba45-360bb4822f7c)


sudo curl -L "https://github.com/docker/compose/releases/download/$COMVER/docker-compose-$(uname -s)-$(uname -m)" -o /usr/bin/docker-compose && \
![Screenshot_2](https://github.com/user-attachments/assets/3aa7f941-2c23-4e11-91cd-96c0a09c22a8)




* 
