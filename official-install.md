# Open Distro for Elasticsearch for Kubernetes


## Official Installation

https://opendistro.github.io/for-elasticsearch-docs/docs/install/helm/

* `helm >= 3`

### Install via `helm`

```bash
git clone https://github.com/opendistro-for-elasticsearch/opendistro-build \
  -b v1.13.0
cd opendistro-build/helm/opendistro-es/

helm package .
# Successfully packaged chart and saved it to: /home/pydemia/git/elasticsearch/opendistro/opendistro-build/helm/opendistro-es/opendistro-es-1.13.0.tgz

NAMESPACE="elasticsearch"
helm install elasticsearch opendistro-es-1.13.0.tgz \
  --namespace=${NAMESPACE} \
  --create-namespace
#   --values=customvalues.yaml

# helm uninstall elasticsearch \
#   --namespace=${NAMESPACE}
```

### Test

#### Test from `elasticsearch` pod inside

```bash
kubectl -n ${NAMESPACE} exec -it elasticsearch-opendistro-es-master-0 -- /bin/bash
```

Then,

```bash
$ curl -XGET https://localhost:9200 -u 'admin:admin' --insecure

{
  "name" : "elasticsearch-opendistro-es-master-0",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "xxx",
  "version" : {
    "number" : "7.10.2",
    "build_flavor" : "oss",
    "build_type" : "tar",
    "build_hash" : "xxx",
    "build_date" : "2021-01-13T00:42:12.435326Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

```bash
curl -k -XGET https://localhost:9200/_cat/indices -u 'admin:admin'
```

#### Test from `local`

* elasticsearch

```bash
kubectl port-forward -n ${NAMESPACE} svc/elasticsearch-opendistro-es-client-service 9200
kubectl port-forward -n ${NAMESPACE} svc/elasticsearch-opendistro-es-data-svc 9200
curl localhost:9200/_cat/indices
```

* kibana

```bash
kubectl port-forward -n ${NAMESPACE} svc/kibana-kibana 5601

curl localhost:5601/_cat/indices
```

---