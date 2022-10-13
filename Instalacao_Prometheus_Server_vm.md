#  Instalção do Prometheus, Grafana e Loki

## Instalação do Prometheus Server usando Binário

### Atualizar o  repositóio de pacotes e atualizar o sistema

```bash
sudo apt update && sudo apt upgrade -y
```
### Baixar o Binário
A versão mais recente pode ser encontrada em https://prometheus.io/download/#prometheus
Durante a escrita deste Howto a versão mais atual é a [**2.39.1**](https://github.com/prometheus/prometheus/releases/download/v2.39.1/prometheus-2.39.1.linux-amd64.tar.gz)  mas para uso em produção e manter o documento mais tempo de suporte será utilizada a versão [**LTS 2.37.1**](https://github.com/prometheus/prometheus/releases/download/v2.37.1/prometheus-2.37.1.linux-amd64.tar.gz)

```bash
curl -LO https://github.com/prometheus/prometheus/releases/download/v2.37.1/prometheus-2.37.1.linux-amd64.tar.gz --output-dir /tmp

```
Ref.
[curl](https://curl.se/docs/manpage.html) 

### Criar usuário de grupo de serviço para gerenciar o prometheus

```bash
sudo addgroup --system prometheus
sudo adduser --shell /sbin/nologin --system --group prometheus
```

Ref. [Comandos para manipulação de contas](https://www.guiafoca.org/guiaonline/inicianteintermediario/ch12.html)

### Criar estruturas de diretórios e mover os arquivos e binários do prometheus
#### Diretorio de configuração 
```
sudo mkdir -p /etc/prometheus
``` 
#### Diretório de dados
```
sudo mkdir -p /var/lib/prometheus
``` 
#### Diretório de logs
```
sudo mkdir -p /var/log/prometheus
```
#### Extrair os arquivos e binários do prometheus e mover para os direótios definitivos
```
tar -zxvf /tmp/prometheus-*.tar.gz -C /tmp
```
#### Mover o binários prometheus e promtool  do para  /usr/local/bin/
```bash
sudo mv /tmp/prometheus-2.37.1.linux-amd64/{prometheus,promtool} /usr/local/bin/
sudo mv /tmp/prometheus-2.37.1.linux-amd64/{consoles,console_libraries} /etc/prometheus/
```

#### Criar configuração inicial do prometheus
Ref.  [Configuração do prometheus](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

Criar arquivo inicial de configuração  /etc/prometheus/prometheus.yml:

```bash

sudo bash -c ' cat <<FIM > /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s 
  evaluation_interval: 15s  
  scrape_timeout : 10s 

scrape_configs:
 
  - job_name: "prometheus"
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:9090"] 
FIM'
```
#### Abaixo segue um conteudo do aquirvo prometheus.yml com os mesmos valores **ativos** com comentários para entender cada um dos parâmtetros. 

```yml
# Configurações contidas no bloco *global* serão utilizadas em todos os jobs a nã ser que haja uma configuração específica no bloco do job.
global:
  scrape_interval: 15s # Definie o intervalo de coleta de dados. O padrão é a cada 1 minuto.
  evaluation_interval: 15s # Intervalo de tempos usado para verificar as regras de alertas. O padrão é a cada 1 minuto.
  scrape_timeout : 10s # Intervalo de tempo usado para condiderar que o alvo está indisponível.

# Configrações do Gerenciador de Alertas
#alerting:
#  alertmanagers:
#    - static_configs:
#        - targets:
          # - alertmanager:9093

#  
## Bloco rules_files é usado para definições de regras de alertas, as regras são carregadas uma vez e em intervalos peródivcos conforme o parâmetro evaluation_interval definido no bloco global 
#rule_files: 
  # - "first_rules.yml"
  # - "second_rules.yml"

# Bloco scrape_configs as configurações dos endpoints de coleta e a maneira que irá coletá-las.
scrape_configs:

    - job_name: "prometheus" # Nome do serviço dado ao prometheus para monitorar usado como rótuo (label) para coleta para qualquer série temporal extraída desta configuração
    scrape_interval: 5s # Definie o intervalo de coleta de dados específico para este job

    # O parâmetro metrics_path define o caminho para acesso as metricas o padrão é '/metrics'
    #metrics_path : "/metrics"
    # O parâmetro scheme define o protocolo usado para as requisições o padrão é http
    #scheme defaults: "http".

    static_configs:
      - targets: ["localhost:9090"] # Endpoint do alvo para coleta, neste caso o próprio prometheus


```

#### Mudar a propriedade dos aquivos e dieretórios do prometheus para o user prometheus

```
sudo chown -R prometheus:prometheus /var/log/prometheus /etc/prometheus \
                                    /var/lib/prometheus /usr/local/bin/{prometheus,promtool}

```
#### Criando um serviço do systemd para o prometheus

```bash
sudo vim /etc/systemd/system/prometheus.service
```

```ini
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecReload=/bin/kill -HUP \$MAINPID
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
 
SyslogIdentifier=prometheus
Restart=always

[Install]
WantedBy=multi-user.target  

```

```ini

[Unit] # Bloco de definição do serviço.
Description=Prometheus # Nome do serviço
Documentation=https://prometheus.io/docs/introduction/overview/ # Documentação do serviço.
Wants=network-online.target # Requisito para ser iniciado, no caso o serviço de rede precisa estar ativo.
After=network-online.target # Serviço será inciado somente após o serviço de rede.

[Service] # Bloco de definição de inicialização do serviço.
Type=simple # Tipo do serviço, o padrão é simple. Pode conter  subserviços.
User=prometheus # Usuário do serviço
Group=prometheus # Grupo do serviço
ExecReload=/bin/kill -HUP \$MAINPID # O serviço do Prometheus será reiniciado ao receber um sinal de reinicialização.
ExecStart=/usr/local/bin/prometheus \ # Comando para o serviço  ser iniciado
  --config.file=/etc/prometheus/prometheus.yml \ 
  --storage.tsdb.path=/var/lib/prometheus \ 
  --web.console.templates=/etc/prometheus/consoles \ 
  --web.console.libraries=/etc/prometheus/console_libraries \ 
  --web.listen-address=0.0.0.0:9090 \ 
  
```

```sh
sudo systemctl daemon-reload

sudo systemctl start prometheus

sudo systemctl enable prometheus

sudo systemctl status prometheus

```
Caso possua firewall ativado liberar o trafego ` sudo ufw allow 9090/tcp`

Acessar o Servidor ` http://<ip address>or<hostname>:9090 `


## Instalando Grafana

### Instalando requisitos necessários 

```sh
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
```

### Adicionando Repositório do grafana OSS

```sh
sudo wget -q -O /usr/share/keyrings/grafana.key https://packages.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

```

### Instalação do grafana via apt 
```sh
sudo apt-get update
sudo apt-get install grafana -y
```

### Ativando o serviço do grafana

```sh
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl status grafana-server
sudo systemctl enable grafana-server.service

```


## Coletando métricas 

### Coletando Metricas do Node

Ref. 
https://prometheus.io/docs/guides/node-exporter/
https://jaanhio.me/blog/linux-node-exporter-setup/

```bash
curl -LO  https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz --output-dir /tmp
tar -xf /tmp/node_exporter-1.4.0.linux-amd64.tar.gz -C /tmp
sudo mv /tmp/node_exporter-1.4.0.linux-amd64/node_exporter  /usr/local/bin
```
```bash
sudo useradd -m node_exporter

sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter


sudo groupadd node_exporter
sudo usermod -a -G node_exporter node_exporter
```
#### Criando serviço  do node_exporter

sudo vim /etc/systemd/system/node_exporter.service

```ini
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target

```bash
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```

#### Adicione o job no bloco do scrape_configs no arquivo promethues.yml no servidor do prometheus

```yml
- job_name: node
  static_configs:
  - targets: ['<IP-DO-ALVO>:9100']
```
### Dashboard para visualizar metricas do node no grafana

https://grafana.com/grafana/dashboards/1860-node-exporter-full/

### Coletando métricas do Nginx 

#### Instale o Nginx 
```
sudo apt install nginx -y
```

Ref.
1. https://igunawan.com/how-to-monitor-basic-metrics-from-nginx-with-prometheus-and-grafana/
2. https://github.com/nginxinc/nginx-prometheus-exporter
3. https://www.observability.blog/nginx-monitoring-with-prometheus/

#### Configuração e ativação do module stup_status do Nginx

Adicione a configuração abaixo no /etc/nginx/nginx.conf no bloco http{}

```nginx
server {
        listen localhost:8080;
        location /metrics {
                stub_status on;
                allow 127.0.0.1;
                deny all;

        }
}

```
Reinicie o ngnix

```bash
sudo service nginx restart
```
As métricas podem ser vistas no localhost usando `curl http://localhost:8080/metrics` .
```output
Active connections: 2 
server accepts handled requests
 23 23 64 

```
#### Instalação e configuração do exporter do nginx para o prometheus

```sh
curl -LO  https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v0.11.0/nginx-prometheus-exporter_0.11.0_linux_amd64.tar.gz --output-dir /tmp
tar -xf /tmp/nginx-prometheus-exporter_0.11.0_linux_amd64.tar.gz -C /tmp
sudo mv /tmp/nginx-prometheus-exporter /usr/local/bin
```
Crie um usuário de serviço par o nginx exporter 

```sh
sudo useradd nginx_exporter
sudo chown nginx_exporter:nginx_exporter /usr/local/bin/nginx-prometheus-exporter


sudo groupadd nginx_exporter
sudo usermod -a -G nginx_exporter nginx_exporter

```

```sh
sudo vim  /etc/systemd/system/nginx_prometheus_exporter.service

```

```ini	
[Unit]
Description=NGINX Prometheus Exporter
After=network.target

[Service]
Type=simple
User=nginx_exporter
Group=nginx_exporter
ExecStart=/usr/local/bin/nginx-prometheus-exporter \
    --web.listen-address=0.0.0.0:9003 \
    --nginx.scrape-uri http://127.0.0.1:8080/metrics

SyslogIdentifier=nginx_prometheus_exporter
Restart=always

[Install]
WantedBy=multi-user.target

```
#### Habilitando o serviço do nginx exporter

```sh
sudo systemctl daemon-reload
sudo systemctl restart nginx_prometheus_exporter
sudo systemctl enable nginx_prometheus_exporter
sudo systemctl status nginx_prometheus_exporter
```
As métricas disponíveis pelo exporter podem ser vistas no localhost usando `curl localhost:9003/metrics` . 
```out
# HELP nginx_connections_accepted Accepted client connections
# TYPE nginx_connections_accepted counter
nginx_connections_accepted 33
# HELP nginx_connections_active Active client connections
# TYPE nginx_connections_active gauge
nginx_connections_active 1
# HELP nginx_connections_handled Handled client connections
# TYPE nginx_connections_handled counter
nginx_connections_handled 33
# HELP nginx_connections_reading Connections where NGINX is reading the request header
# TYPE nginx_connections_reading gauge
nginx_connections_reading 0
# HELP nginx_connections_waiting Idle client connections
# TYPE nginx_connections_waiting gauge
nginx_connections_waiting 0
# HELP nginx_connections_writing Connections where NGINX is writing the response back to the client
# TYPE nginx_connections_writing gauge
nginx_connections_writing 1
# HELP nginx_http_requests_total Total http requests
# TYPE nginx_http_requests_total counter
nginx_http_requests_total 94
# HELP nginx_up Status of the last metric scrape
# TYPE nginx_up gauge
nginx_up 1
# HELP nginxexporter_build_info Exporter build information
# TYPE nginxexporter_build_info gauge
nginxexporter_build_info{arch="linux/amd64",commit="e4a6810d4f0b776f7fde37fea1d84e4c7284b72a",date="2022-09-07T21:09:51Z",dirty="false",go="go1.19",version="0.11.0"} 1
```

#### Configurando o job no prometheus para coleta do nginx

Adicione no bloco scrape_configs do arquivo /etc/prometheus/prometheus.yml as configurações abaixo:

```yml
  - job_name: server_nginx
    scrape_interval: 5s
    static_configs:
      - targets: ['0.0.0.0:9003']
        labels:
          alias: nginx_server
```
#### Reinicie o prometheus 

```sh
sudo systemctl restart prometheus 
sudo systemctl status prometheus

``` 
#### Dashboard para visualizar as metricas 
Baixe o arquivo json  e import no grafana

```sh
curl -LO https://raw.githubusercontent.com/nginxinc/nginx-prometheus-exporter/main/grafana/dashboard.json
```
## Instalação do Loki

### Instalação do pacotes e configuração do Loki

```
 sudo apt install unzip
 curl -LO "https://github.com/grafana/loki/releases/download/v2.6.1/loki-linux-amd64.zip" --output-dir /tmp
  unzip /tmp/loki-linux-amd64.zip -d /tmp
 sudo mv /tmp/loki-linux-amd64 /usr/local/bin
 sudo  chmod a+x /usr/local/bin/loki-linux-amd64
 wget https://raw.githubusercontent.com/grafana/loki/v2.6.1/cmd/loki/loki-local-config.yaml
 sudo mv /tmp/loki-local-config.yaml /usr/local/bin/

```
### Executando Loki como serviço 

```sh

sudo useradd -m loki
sudo groupadd loki
sudo usermod -a -G loki loki
sudo chown loki:loki /usr/local/bin/loki-linux-amd64

```
### Arquivo de inicialização do Loki (loki.service)

```sh
sudo vim  /etc/systemd/system/loki.service
```
```ini 

[Unit]
Description=Loki service
After=network.target

[Service]
Type=simple
User=loki
ExecStart=/usr/local/bin/loki-linux-amd64 -config.file /usr/local/bin/loki-local-config.yaml

[Install]
WantedBy=multi-user.target

``` 
#### Ative o serviço loki
```
sudo service loki start
sudo service loki status
sudo service loki enable

```
#### Teste o acesso 

`curl localhost:3100/metrics`


## Instalação do Promtail 

### Instalação do pacotes e configuração do Promtail

```sh
 sudo apt install unzip
 curl -LO "https://github.com/grafana/loki/releases/download/v2.6.1/promtail-linux-amd64.zip" --output-dir /tmp
 
 unzip promtail-linux-amd64.zip -d /tmp
 sudo mv /tmp/promtail-linux-amd64 /usr/local/bin
 sudo  chmod a+x /usr/local/bin/promtail-linux-amd64
 wget https://raw.githubusercontent.com/grafana/loki/master/clients/cmd/promtail/promtail-local-config.yaml
 sudo mv promtail-local-config.yaml /usr/local/bin/

```
### Executando promtail como serviço 

```sh

sudo useradd -m promtail
sudo groupadd promtail
usermod -a -G adm promtail
sudo chown promtail:promtail /usr/local/bin/promtail-linux-amd64

```
### Arquivo de inicialização do promtail (promtail.service)
 
```
sudo vim /etc/systemd/system/promtail.service
```

```ini 

[Unit]
Description=promtail service
After=network.target

[Service]
Type=simple
User=promtail
ExecStart=/usr/local/bin/promtail-linux-amd64 -config.file /usr/local/bin/promtail-local-config.yaml

[Install]
WantedBy=multi-user.target

``` 
#### Ative o serviço promtail
```
sudo service promtail start
sudo service promtail status
sudo service promtail enable

```
#### Teste o acesso 

`curl localhost:9080/metrics`


