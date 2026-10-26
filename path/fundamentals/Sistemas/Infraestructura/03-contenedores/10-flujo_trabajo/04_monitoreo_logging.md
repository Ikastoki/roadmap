# Monitoreo y logging

Los contenedores deben tratar stdout y stderr como los canales unicos de logging. No deben guardar logs en archivos locales dentro del contenedor. Los agentes de logging centralizan los logs de todos los contenedores en un sistema de busqueda y analisis. Las metricas de recursos se recopilan mediante exporters especializados.

## Logging con stdout/stderr

```dockerfile
# BIEN: la app escribe a stdout
# Node.js
console.log("Servidor iniciado en puerto 3000");
console.error("Error de conexion a base de datos");

# Python
import logging
logging.info("Servidor iniciado")
logging.error("Error de conexion")
```

```bash
# Ver logs de un contenedor
docker logs mi-app
docker logs -f mi-app  # Seguir en tiempo real
docker logs --tail 100 mi-app  # Ultimas 100 lineas
```

## Stack ELK (Elasticsearch, Logstash, Kibana)

```yaml
# docker-compose.yml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  app:
    image: mi-app
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  es-data:
```

## Prometheus y Grafana

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"

volumes:
  prometheus-data:
  grafana-data:
```

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

## Configuracion de logging

```yaml
# En Docker Compose
services:
  app:
    image: mi-app
    logging:
      driver: json-file
      options:
        max-size: "10m"    # Tamano maximo por archivo
        max-file: "3"      # Numero de archivos a rotar
        tag: "{{.Name}}"

  # Enviar a un servicio de logging centralizado
  app-syslog:
    image: mi-app
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://logserver:514"
```

> El monitoreo y logging en contenedores siguen el mismo patron que en servidores tradicionales, pero adaptado a la naturaleza efimera de los contenedores: los logs van a stdout/stderr y los agentes los recogen; las metricas las recogen exporters que conectan al motor de contenedores. La clave es no intentar guardar nada dentro del contenedor.

