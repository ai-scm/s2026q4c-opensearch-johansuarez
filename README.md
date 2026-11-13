# Actividad OpenSearch

Documentación de los pasos seguidos en la práctica con OpenSearch y OpenSearch Dashboards.

## Parte 1

### 1. Instalación local con Docker Compose

#### 1.1 Crear el archivo `docker-compose.yml`

Se usa el siguiente archivo para el levantamiento:

```yaml
services:
  opensearch-node1:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node1
    environment:
      - cluster.name=opensearch-cluster # Name the cluster
      - node.name=opensearch-node1 # Name the node that will run in this container
      - discovery.seed_hosts=opensearch-node1,opensearch-node2 # Nodes to look for when discovering the cluster
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2 # Nodes eligibile to serve as cluster manager
      - bootstrap.memory_lock=true # Disable JVM heap memory swapping
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # Set min and max JVM heap sizes to at least 50% of system RAM
      - "DISABLE_INSTALL_DEMO_CONFIG=true" # Prevents execution of bundled demo script which installs demo certificates and security configurations to OpenSearch
      - "DISABLE_SECURITY_PLUGIN=true" # Disables Security plugin
    ulimits:
      memlock:
        soft: -1 # Set memlock to unlimited (no soft or hard limit)
        hard: -1
      nofile:
        soft: 65536 # Maximum number of open files for the opensearch user - set to at least 65536
        hard: 65536
    volumes:
      - opensearch-data1:/usr/share/opensearch/data # Creates volume called opensearch-data1 and mounts it to the container
    ports:
      - 9200:9200 # REST API
    networks:
      - opensearch-net # All of the containers will join the same Docker bridge network
  opensearch-node2:
    image: opensearchproject/opensearch:latest
    container_name: opensearch-node2
    environment:
      - cluster.name=opensearch-cluster # Name the cluster
      - node.name=opensearch-node2 # Name the node that will run in this container
      - discovery.seed_hosts=opensearch-node1,opensearch-node2 # Nodes to look for when discovering the cluster
      - cluster.initial_cluster_manager_nodes=opensearch-node1,opensearch-node2 # Nodes eligibile to serve as cluster manager
      - bootstrap.memory_lock=true # Disable JVM heap memory swapping
      - "OPENSEARCH_JAVA_OPTS=-Xms512m -Xmx512m" # Set min and max JVM heap sizes to at least 50% of system RAM
      - "DISABLE_INSTALL_DEMO_CONFIG=true" # Prevents execution of bundled demo script which installs demo certificates and security configurations to OpenSearch
      - "DISABLE_SECURITY_PLUGIN=true" # Disables Security plugin
    ulimits:
      memlock:
        soft: -1 # Set memlock to unlimited (no soft or hard limit)
        hard: -1
      nofile:
        soft: 65536 # Maximum number of open files for the opensearch user - set to at least 65536
        hard: 65536
    volumes:
      - opensearch-data2:/usr/share/opensearch/data # Creates volume called opensearch-data2 and mounts it to the container
    networks:
      - opensearch-net # All of the containers will join the same Docker bridge network
  opensearch-dashboards:
    image: opensearchproject/opensearch-dashboards:latest
    container_name: opensearch-dashboards
    ports:
      - 5601:5601 # Map host port 5601 to container port 5601
    expose:
      - "5601" # Expose port 5601 for web access to OpenSearch Dashboards
    environment:
      - 'OPENSEARCH_HOSTS=["http://opensearch-node1:9200","http://opensearch-node2:9200"]'
      - "DISABLE_SECURITY_DASHBOARDS_PLUGIN=true" # disables security dashboards plugin in OpenSearch Dashboards
    networks:
      - opensearch-net

volumes:
  opensearch-data1:
  opensearch-data2:

networks:
  opensearch-net:
```

#### 1.2 Levantar los contenedores

```bash
docker compose up -d
```

#### 1.3 Verificar que los contenedores estén activos

```bash
docker compose ps
```

#### 1.4 Verificar que OpenSearch responda correctamente

```bash
curl http://localhost:9200
```

#### 1.5 Acceder a OpenSearch Dashboards

Abrir en el navegador: <http://localhost:5601/>

---

### 2. Carga de Sample Data

Desde la página de inicio de OpenSearch Dashboards se agregó el conjunto de datos de ejemplo **Sample web logs**, que crea el índice `opensearch_dashboards_sample_data_logs`.

---

### 3. Inserción manual de un documento (Dev Tools)

En **Dev Tools** se agregó un documento adicional al índice de ejemplo:

```json
PUT opensearch_dashboards_sample_data_logs/_doc/log-manual-001
{
  "agent": "Mozilla/5.0",
  "bytes": 2048,
  "clientip": "192.0.2.10",
  "extension": "html",
  "host": "www.ejemplo.com",
  "message": "Registro creado manualmente para la practica",
  "request": "/practica.html",
  "response": 200,
  "timestamp": "2026-11-13T17:30:00Z",
  "url": "https://www.ejemplo.com/practica.html"
}
```

### 4. Verificación del documento

```json
GET opensearch_dashboards_sample_data_logs/_doc/log-manual-001
```

---

### 5. Visualizaciones

Se crearon 3 visualizaciones sobre el índice de logs, todas agrupadas por intervalo de **3 horas** sobre el campo `timestamp`:

| Visualización | Tipo | Descripción |
|---|---|---|
| **(Line) Avg bytes over time** | Línea | Promedio del campo `bytes` a lo largo del tiempo (eje Y: *Average bytes*, eje X: *timestamp per 3 hours*). |
| **(Vertical Bar) Stacked responses over time** | Barras verticales apiladas | Conteo de peticiones por intervalo de tiempo, apiladas por código de respuesta (`response`): **200**, **503** y **404**. |
| **(Metric) Unique visitors** | Métrica | Conteo único del campo `clientip` (*Unique count of clientip*). Sin filtros, el resultado fue **820**. |

<!-- Captura: docs/img/visualizaciones.png -->

---

### 6. Faceted Search

Se utilizó el panel **[Logs] Input Controls**, que ofrece tres controles de filtrado: **Source Country**, **OS** y **Bytes** (rango de 0 a 19837). Al aplicar los filtros, las tres visualizaciones se actualizan en conjunto.

| Source Country | OS | Bytes | Unique visitors | Respuestas observadas |
|---|---|---|---|---|
| US | ios | Sin filtro (0 – 19837) | **22** | 200 y 404 |
| US | win xp | 1 – 8000 | **16** | 200 |
| MX | win 7 | Sin filtro (0 – 19837) | **7** | 200 |


<!-- Capturas: docs/img/faceted-1.png, docs/img/faceted-2.png, docs/img/faceted-3.png -->

---

## Parte 2: Exploración del API REST de OpenSearch

Además de usar OpenSearch Dashboards, se puede consultar el motor directamente por su **API REST**, de dos formas equivalentes:

- **Línea de comandos** con `curl` (por ejemplo contra `http://localhost:9200`).
- **Dev Tools** (consola de desarrollo) dentro de Dashboards, donde se escribe `MÉTODO /ruta` y, opcionalmente, un cuerpo JSON.

Por el momento, solo usé Dev Tools, pero se dejan los respectivos comandos que irían en la consola.


---

### 2.1 Consultas básicas

#### Obtener un solo documento (`_search` con `size: 1`)

Desde la línea de comandos:

```bash
curl -X GET "http://localhost:9200/opensearch_dashboards_sample_data_logs/_search?size=1"
```

Equivalente en Dev Tools:

```json
GET opensearch_dashboards_sample_data_logs/_search
{
  "size": 1
}
```


#### Contar documentos (`_count`)

```bash
curl -X GET "http://localhost:9200/opensearch_dashboards_sample_data_logs/_count"
```

```json
GET opensearch_dashboards_sample_data_logs/_count
```

#### Ver la estructura del índice (`_mapping`)

```bash
curl -X GET "http://localhost:9200/opensearch_dashboards_sample_data_logs/_mapping"
```

```json
GET opensearch_dashboards_sample_data_logs/_mapping
```

#### Ver la configuración del índice (`_settings`)

```bash
curl -X GET "http://localhost:9200/opensearch_dashboards_sample_data_logs/_settings"
```

```json
GET opensearch_dashboards_sample_data_logs/_settings
```

#### Buscar el documento insertado manualmente

```json
GET opensearch_dashboards_sample_data_logs/_search
{
  "query": {
    "match": {
      "message": "Registro creado manualmente para la practica"
    }
  }
}
```


---

### 2.2 Faceted search con agregaciones


#### Facetas por host y por respuesta

```json
GET opensearch_dashboards_sample_data_logs/_search
{
  "size": 0,
  "aggs": {
    "por_host": {
      "terms": {
        "field": "host.keyword"
      }
    },
    "por_response": {
      "terms": {
        "field": "response.keyword"
      }
    }
  }
}
```

#### Promedio de bytes por hora del día

```json
GET opensearch_dashboards_sample_data_logs/_search
{
  "size": 0,
  "aggs": {
    "bytes_en_el_tiempo": {
      "date_histogram": {
        "field": "timestamp",
        "fixed_interval": "3h"
      },
      "aggs": {
        "promedio_bytes": {
          "avg": { "field": "bytes" }
        }
      }
    }
  }
}
```

#### Respuestas por intervalo de tiempo

```json
GET opensearch_dashboards_sample_data_logs/_search
{
  "size": 0,
  "aggs": {
    "respuestas_en_el_tiempo": {
      "date_histogram": {
        "field": "timestamp",
        "fixed_interval": "3h"
      },
      "aggs": {
        "por_response": {
          "terms": { "field": "response.keyword" }
        }
      }
    }
  }
}
```
