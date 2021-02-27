promstack-exporters
========================

A monitoring client for [Promstack](https://github.com/paaacman/promstack).  
This project includes [node-exporter](https://github.com/prometheus/node_exporter), [Cadvisor](https://github.com/google/cadvisor), [nginx-exporter](https://github.com/nginxinc/nginx-prometheus-exporter) and [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/).

## Install
First create a `.env` file from [.env.dist](.env.dist).
- `HOST_NAME` the name of the host where promstack-exporters is installed. This will be available as a label in scraped data. 
- `ENV` the env of the host where promstack-exporters is installed. This will be available as a label in promtail data. 
- `NGINX_SCRAPE_URI` the nginx stub_status URL, see [nginx-exporter section](#nginx-exporter)
- `DOCKER_HOST_INTERNAL_IP` the internal IP of docker host, used by nginx-exporter to access host's nginx stub_page.
- `LOKI_EXTERNAL_URL` Loki URL to be called by Promtail, see [Promtail](#promtail) bellow.  
- `LOKI_USERNAME` Loki username for Promtail basic auth to Loki.  
- `LOKI_PASSWORD` Loki password for Promtail basic auth to Loki.  

Then up the project with docker-compose:
```bash
git clone git@github.com:paaacman/promstack-exporters.git
cd promstack-exporters
docker-compose up -d
```

### Containers

* node-exporter (host metrics collector, send data to Prometheus)
* cAdvisor (containers metrics collector, sent to Prometheus)
* nginx-exporter (collect nginx status and send it to Prometheus)
* promtail (get log files and send it to Loki)

## node-exporter
> Prometheus exporter for hardware and OS metrics exposed by *NIX kernels, written in Go with pluggable metric collectors.

Metrics are scraped by Prometheus, container needs to be accessible by Prometheus.

See [node-exporter documentation](https://github.com/prometheus/node_exporter).

## Cadvisor
Cadvisor monitor containers, like docker containers.  
Metrics are scraped by Prometheus, container needs to be accessible by Prometheus.   

See [Cadvisor documentation](https://github.com/google/cadvisor).  
  
## nginx-exporter
nginx-exporter uses [nginx status page](http://nginx.org/en/docs/http/ngx_http_stub_status_module.html#stub_status) to get data about nginx usage. Those data can then be scraped by Prometheus.
Metrics are scraped by Prometheus, container needs to be accessible by Prometheus.   
Nginx stub_status page can be set with the `NGINX_SCRAPE_URI` environment variable. To monitor host's nginx instance, nginx-exporter container have to access this URI on host network by adding this docker-compose configuration:  
```yaml
# docker-compose.yml
services:
  nginxexporter:
    ...
    extra_hosts:
    - "${DOCKER_HOST_NAME:${DOCKER_HOST_INTERNAL_IP}"
    ...
```
  
See [nginx-exporter](https://github.com/nginxinc/nginx-prometheus-exporter) for more information.


## Promtail
According to [the doc](https://grafana.com/docs/loki/latest/clients/promtail/):
> Promtail is an agent which ships the contents of local logs to a private Loki instance or Grafana Cloud. It is usually deployed to every machine that has applications needed to be monitored.

It sends logs to your Loki instance (available in [Promstack project](https://github.com/paaacman/promstack)) at the URL given in `LOKI_EXTERNAL_URL` environment variable.
In this promstack-exporters project, Promtail is configured to log to Loki with basic auth, provided by a nginx reverse proxy. See [nginx documentation](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/) for more information.
  
Adapt the `scrape_configs` in [promtail.yml](promtail/promtail.yml) to your needs.  
  
See [Promtail documentation](https://grafana.com/docs/loki/latest/clients/promtail/) for more information.

### Backup
To backup Promtail current state to `/tmp/promtail_backup`, you can backup `promtail_data` volume like this:
```bash
PROMTAIL_VOLUME_ID=$(docker volume ls -q | grep promtail_data)
docker run --rm --volume "$PROMTAIL_VOLUME_ID:/promtail" --volume "/tmp/promtail_backup:/backup"  debian  tar cpf "/backup/promtail-volume-backup.tar" -C "/" "promtail"
```
  
See this technic to restore the volume: [https://github.com/fjh1997/docker_named_volume_backup](https://github.com/fjh1997/docker_named_volume_backup).  
