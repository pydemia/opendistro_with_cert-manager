# Open Distro for Elasticsearch for Kubernetes

## Customized Installation

* Set certs with `cert-manager`
* Use pre-defined user `kibanaserver` as a service account let kibana to access to elasticsearch

### Fork the official helm chart

```bash
git clone https://github.com/opendistro-for-elasticsearch/opendistro-build \
  -b v1.13.0
cp -rf ./opendistro-build/helm/opendistro-es opendistro-es-custom
cd opendistro-es-custom
```

### Preparation

#### Namespace

```bash
NAMESPACE="elasticsearch"
kubectl create -n ${NAMESPACE}
```

#### Set ServiceAccount as a secret

```bash
kubectl -n ${NAMESPACE} apply -f opendistro-es-user.yaml
```

or

```bash
kubectl -n ${NAMESPACE} create secret generic kibanaserver-user \
  --from-literal=username=kibanaserver \
  --from-literal=password=kibanaserver \
  --from-literal=cookie="$(date +%s | sha256sum | head -c 128; echo)"
```

* username: `kibanaserver` as default
* password: `kibanaserver` as default
* cookie: a cookie secret, which is a string has a minimum length of **_32_** character.
  * In this case, I use bytes to create a random string.
    `utf8` uses 1~4 bytes to represent a character. Then I got a 32-length-guaranteed-string with `head -c 32x4=128`(`head -c` option counts character in byte!). You can use it with any other strings.

See [opendistro-for-elasticsearch/security](https://github.com/opendistro-for-elasticsearch/security/blob/main/securityconfig/internal_users.yml)

#### Set `Issuers` and `Certificates` via cert-manager

We assume that you already set `cert-manager` in your cluster.
If not, follow [cert-manager Installation Guide](https://cert-manager.io/docs/installation/).
We use `cert-manager=v1.1` in this article.

```bash
$ kubectl -n ${NAMESPACE} apply -f opendistro-es-tls.yaml
issuer.cert-manager.io/odfs-elk-ca-issuer created
certificate.cert-manager.io/elasticsearch-ca-certs created
issuer.cert-manager.io/odfs-elk-issuer created
certificate.cert-manager.io/elasticsearch-transport-certs created
certificate.cert-manager.io/elasticsearch-rest-certs created
certificate.cert-manager.io/elasticsearch-admin-certs created
certificate.cert-manager.io/kibana-certs created
```

Description:

* Issuer
  * `odfs-elk-ca-issuer`: issuer for a RSA key pair to generate `root ca`.
  * `odfs-elk-issuer`: issuer for a RSA key pair to generate `ssl` under `root ca`.
* Certificates
  * `elasticsearch-ca-certs`: defines a RSA key pair spec and actually generate a ca key pair(`ca.crt`==`tls.crt`, `tls.key`) with `secretName`.
  * `elasticsearch-transport-certs`: defines a RSA key pair spec and actually generate a key pair using `ca.crt`, with `secretName`.
  * `elasticsearch-rest-certs`: defines a RSA key pair spec and actually generate a key pair using `ca.crt`, with `secretName`.
  * `elasticsearch-admin-certs`: defines a RSA key pair spec and actually generate a key pair using `ca.crt`, with `secretName`.
  * `kibana-certs`: defines a RSA key pair spec and actually generate a key pair using `ca.crt`, with `secretName`.

* Secrets
  * `elasticsearch-ca-certs`: **PKCS#8** RSA key pair [`ca.crt`==`tls.crt`, `tls.key`]
  * `elasticsearch-transport-certs`: **PKCS#8** RSA key pair [`ca.crt`, `tls.crt`, `tls.key`]
  * `elasticsearch-rest-certs`: **PKCS#8** RSA key pair [`ca.crt`, `tls.crt`, `tls.key`]
  * `elasticsearch-admin-certs`: **PKCS#8** RSA key pair [`ca.crt`, `tls.crt`, `tls.key`]
  * `kibana-certs`: **PKCS#8** RSA key pair [`ca.crt`, `tls.crt`, `tls.key`]


:warning: Secret Requirements

* It should be `encoding: PKCS8`
* `spec.commonName` and `spec.subject` info should be managed and used when validate certificate chain. See [Describe values.yaml](#describe-value-yaml) below.

Check your setup:

```bash
$ kubectl -n ${NAMESPACE} get secrets
NAME                            TYPE                                  DATA   AGE
default-token-xxxxx             kubernetes.io/service-account-token   3      93s
elasticsearch-admin-certs       kubernetes.io/tls                     3      71s
elasticsearch-ca-certs          kubernetes.io/tls                     3      74s
elasticsearch-rest-certs        kubernetes.io/tls                     3      71s
elasticsearch-transport-certs   kubernetes.io/tls                     3      72s
kibana-certs                    kubernetes.io/tls                     3      70s
kibanaserver-user               Opaque                                3      83s
```

### Install via `helm`, with custom `values.yaml`

* `values.yaml`
  * `opendistro-es.yaml`
  * `opendistro-kibana.yaml`

* For `bash` and `zsh`: ()

```bash
RELEASE_NAME="elasticsearch"
helm install ${RELEASE_NAME} opendistro-es-1.13.0.tgz \
  --namespace=${NAMESPACE} \
  --values=opendistro-es.yaml \
  --values=opendistro-kibana.yaml \
  --set kibana.extraEnvs\[0\].name="ELASTICSEARCH_HOSTS" \
  --set kibana.extraEnvs\[0\].value="https://${RELEASE_NAME}-opendistro-es-client-service:9200"

# For zsh, use `\[0\]` instead of `[0]` avoid this error: zsh: no matches found: kibana.extraEnvs[0].name=...

# helm -n ${NAMESPACE} uninstall ${RELEASE_NAME}

RELEASE_NAME="elasticsearch"
helm upgrade ${RELEASE_NAME} opendistro-es-1.13.0.tgz \
  --namespace=${NAMESPACE} \
  --values=opendistro-es.yaml \
  --values=opendistro-kibana.yaml \
  --set kibana.extraEnvs\[0\].name="ELASTICSEARCH_HOSTS" \
  --set kibana.extraEnvs\[0\].value="https://${RELEASE_NAME}-opendistro-es-client-service:9200"
```

#### Describe `values.yaml`

* Mount certs, changing its key name for internal use.

Ex. `elasticsearch-transport-certs.tls\.crt` -> `/usr/share/elasticsearch/config/elk-transport-crt.pem`

```yaml
elasticsearch:
  ssl:
    ## TLS is mandatory for the transport layer and can not be disabled
    transport:
      existingCertSecret: elasticsearch-transport-certs
      existingCertSecretCertSubPath: tls.crt  # It mounts as elk-transport-crt.pem
      existingCertSecretKeySubPath: tls.key   # It mounts as elk-transport-key.pem
      existingCertSecretRootCASubPath: ca.crt # It mounts as elk-transport-root-ca.pem
    rest:
      enabled: true
      existingCertSecret: elasticsearch-rest-certs
      existingCertSecretCertSubPath: tls.crt  # It mounts as elk-rest-crt.pem
      existingCertSecretKeySubPath: tls.key   # It mounts as elk-rest-key.pem
      existingCertSecretRootCASubPath: ca.crt # It mounts as elk-rest-root-ca.pem
    admin:
      enabled: true
      existingCertSecret: elasticsearch-admin-certs
      existingCertSecretCertSubPath: tls.crt  # It mounts as admin-crt.pem
      existingCertSecretKeySubPath: tls.key   # It mounts as admin-key.pem
      existingCertSecretRootCASubPath: ca.crt # It mounts as admin-root-ca.pem

  configDirectory: /usr/share/elasticsearch/config
```

* This sets `volumes` and `volumemounts` as [the following](https://github.com/opendistro-for-elasticsearch/opendistro-build/blob/main/helm/opendistro-es/templates/elasticsearch/es-master-sts.yaml#L185).

```yaml
        - mountPath: {{ .Values.elasticsearch.configDirectory }}/elk-transport-crt.pem
          name: transport-certs
          subPath: {{ .Values.elasticsearch.ssl.transport.existingCertSecretCertSubPath }}
```

* opendistro/security plugin uses **DN** to validate certificate chain.
Write your own info there, following [this](https://opendistro.github.io/for-elasticsearch-docs/docs/troubleshoot/tls/#validate-certificate-chain).

```yaml
elasticsearch:
  config:
    opendistro_security.authcz.admin_dn:
    - 'CN=elasticsearch.svc.cluster.local,OU=example unit,O=example org.,C=KR'
    - 'CN=*.elasticsearch.svc.cluster.local,OU=example unit,O=example org.,C=KR'
    opendistro_security.nodes_dn:
    - 'CN=elasticsearch.svc.cluster.local,OU=example unit,O=example org.,C=KR'
    - 'CN=*.elasticsearch.svc.cluster.local,OU=example unit,O=example org.,C=KR'
```

* Disable demo-certificate setting(OpenDistro for Elasticsearch Security Demo Installer)

```yaml
elasticsearch:
  config:
    opendistro_security.allow_unsafe_democertificates: false
```

##### `opendistro-es.yaml`

```yaml
elasticsearch:
  ssl:
    ## TLS is mandatory for the transport layer and can not be disabled
    transport:
      existingCertSecret: elasticsearch-transport-certs
      existingCertSecretCertSubPath: tls.crt  # It mounts as elk-transport-crt.pem
      existingCertSecretKeySubPath: tls.key   # It mounts as elk-transport-key.pem
      existingCertSecretRootCASubPath: ca.crt # It mounts as elk-transport-root-ca.pem
    rest:
      enabled: true
      existingCertSecret: elasticsearch-rest-certs
      existingCertSecretCertSubPath: tls.crt  # It mounts as elk-rest-crt.pem
      existingCertSecretKeySubPath: tls.key   # It mounts as elk-rest-key.pem
      existingCertSecretRootCASubPath: ca.crt # It mounts as elk-rest-root-ca.pem
    admin:
      enabled: true
      existingCertSecret: elasticsearch-admin-certs
      existingCertSecretCertSubPath: tls.crt  # It mounts as admin-crt.pem
      existingCertSecretKeySubPath: tls.key   # It mounts as admin-key.pem
      existingCertSecretRootCASubPath: ca.crt # It mounts as admin-root-ca.pem

  config:
  # TLS Configuration Transport Layer
    opendistro_security.ssl.transport.pemcert_filepath: elk-transport-crt.pem
    opendistro_security.ssl.transport.pemkey_filepath: elk-transport-key.pem
    opendistro_security.ssl.transport.pemtrustedcas_filepath: elk-transport-root-ca.pem
    opendistro_security.ssl.transport.enforce_hostname_verification: false
    # opendistro_security.ssl.transport.truststore_filepath: opendistro-es.truststore

    # TLS Configuration REST Layer
    opendistro_security.ssl.http.enabled: true
    opendistro_security.ssl.http.pemcert_filepath: elk-rest-crt.pem
    opendistro_security.ssl.http.pemkey_filepath: elk-rest-key.pem
    opendistro_security.ssl.http.pemtrustedcas_filepath: elk-rest-root-ca.pem

    # 
    opendistro_security.allow_unsafe_democertificates: false
    opendistro_security.allow_default_init_securityindex: true

    # See https://opendistro.github.io/for-elasticsearch-docs/docs/troubleshoot/tls/#validate-certificate-chain
    # It is used to validate certificate chain. Set 
    opendistro_security.authcz.admin_dn:
    - 'CN=elasticsearch.svc.cluster.local,OU=example unit,O=example org.,C=KR'
    - 'CN=*.elasticsearch.svc.cluster.local,OU=example unit,O=example org.,C=KR'
    opendistro_security.nodes_dn:
    - 'CN=elasticsearch.svc.cluster.local,OU=example unit,O=example org.,C=KR'
    - 'CN=*.elasticsearch.svc.cluster.local,OU=example unit,O=example org.,C=KR'

    opendistro_security.enable_snapshot_restore_privilege: true
    opendistro_security.check_snapshot_restore_write_privileges: true
    opendistro_security.restapi.roles_enabled: ["all_access", "security_rest_api_access"]
    opendistro_security.system_indices.enabled: true
    opendistro_security.system_indices.indices: [".opendistro-alerting-config", ".opendistro-alerting-alert*", ".opendistro-anomaly-results*", ".opendistro-anomaly-detector*", ".opendistro-anomaly-checkpoints", ".opendistro-anomaly-detection-state", ".opendistro-reports-*", ".opendistro-notifications-*", ".opendistro-notebooks", ".opendistro-asynchronous-search-response*"]
    cluster.routing.allocation.disk.threshold_enabled: false
    
    # See https://opendistro.github.io/for-elasticsearch-docs/docs/security/audit-logs/storage-types/
    opendistro_security.audit.type: internal_elasticsearch
    opendistro_security.audit.config.disabled_rest_categories: NONE
    opendistro_security.audit.config.disabled_transport_categories: NONE

```

##### `opendistro-kibana.yaml`

* `kibana.config.elasticsearch.requestTimeout` minimum value is `360000`.

```yaml
kibana:
  config:
    elasticsearch.requestTimeout: 360000
```

* set auth. to access es
See [Set ServiceAccount as a secret](#set-serviceaccount-as-a-secret)

```yaml
kibana:
  elasticsearchAccount:
    secret: kibanaserver-user
    keyPassphrase:
      enabled: false
```

The `username`, `password` and `cookie` is referenced as Env. and called as the following:

```yaml
kibana:
  config:
    opendistro_security.cookie.secure: true
    opendistro_security.cookie.password: ${COOKIE_PASS}

    elasticsearch.username: ${ELASTICSEARCH_USERNAME}
    elasticsearch.password: ${ELASTICSEARCH_PASSWORD}
```

* Mount certs, changing its key name for internal use.

Ex. `kibana-certs.tls\.crt` -> `/usr/share/kibana/certs/kibana-crt.pem`

```yaml
kibana:
  ssl:
    kibana:
      enabled: true
      existingCertSecret: kibana-certs
      existingCertSecretCertSubPath: tls.crt  # kibana-crt.pem
      existingCertSecretKeySubPath: tls.key   # kibana-key.pem
      existingCertSecretRootCASubPath: ca.crt # kibana-root-ca.pem
    elasticsearch:
      enabled: true
      existingCertSecret: elasticsearch-rest-certs
      existingCertSecretCertSubPath: tls.crt  # elk-rest-crt.pem
      existingCertSecretKeySubPath: tls.key   # elk-rest-key.pem
      existingCertSecretRootCASubPath: ca.crt # elk-rest-root-ca.pem

  certsDirectory: "/usr/share/kibana/certs"
```

And its usage:

```yaml
kibana:
  config:
    # Kibana TLS Config
    server.ssl.enabled: true
    server.ssl.certificate: /usr/share/kibana/certs/kibana-crt.pem
    server.ssl.key: /usr/share/kibana/certs/kibana-key.pem

    opendistro_security.allow_client_certificates: true
    elasticsearch.ssl.verificationMode: none  # certificate -> ConnectionError raised(It's weird!)
    elasticsearch.ssl.certificate: /usr/share/kibana/certs/elk-rest-crt.pem
    elasticsearch.ssl.key: /usr/share/kibana/certs/elk-rest-key.pem
    elasticsearch.ssl.certificateAuthorities: ["/usr/share/kibana/certs/elk-rest-root-ca.pem"]

    # Multitenancy with global/private tenants disabled,
    # set to both to true if you want them to be available.
    opendistro_security.multitenancy.enabled: true
    opendistro_security.multitenancy.tenants.enable_private: false
    opendistro_security.multitenancy.tenants.enable_global: false
    opendistro_security.readonly_mode.roles: ["kibana_read_only"]
    elasticsearch.requestHeadersWhitelist: ["securitytenant","Authorization"]
  
  configDirectory: "/usr/share/kibana/config"
```

* Set extraEnv `ELASTICSEARCH_HOSTS` in kibana

```bash
  --set kibana.extraEnvs[0].name="ELASTICSEARCH_HOSTS" \
  --set kibana.extraEnvs[0].value="https://${RELEASE_NAME}-opendistro-es-client-service:9200"
```

See [this](https://github.com/opendistro-for-elasticsearch/opendistro-build/blob/main/helm/opendistro-es/templates/kibana/kibana-deployment.yaml#L56).  
As it said, if no custom configuration provided, default to internal DNS.  
It means if helm release name is `elasticsearch`, helm template "opendistro-es.fullname" is `elasticsearch-opendistro-es`, then the hostname would be `elasticsearch-opendistro-es-client-service`.

This URL depends on what your helm `${RELEASE_NAME}` is, so we should inject this in external way `"https://${RELEASE_NAME}-opendistro-es-client-service:9200"`.

##### Further troubleshooting points

`kibana.config.elasticsearch.ssl.verificationMode: none`: Why not working with `certificate`?

* When set `certificate`: ConnectionError?

```log
│ {"type":"log","@timestamp":"2021-02-24T18:28:27Z","tags":["debug","metrics"],"pid":1,"message":"Refreshing metrics"}
│ {"type":"log","@timestamp":"2021-02-24T18:28:28Z","tags":["error","elasticsearch","data"],"pid":1,"message":"[ConnectionError]: self signed certificate"}
│ {"type":"log","@timestamp":"2021-02-24T18:28:31Z","tags":["error","elasticsearch","data"],"pid":1,"message":"[ConnectionError]: self signed certificate"}
│ {"type":"log","@timestamp":"2021-02-24T18:28:32Z","tags":["debug","metrics"],"pid":1,"message":"Refreshing metrics"}
│ {"type":"log","@timestamp":"2021-02-24T18:28:33Z","tags":["error","elasticsearch","data"],"pid":1,"message":"[ConnectionError]: self signed certificate"}
│ {"type":"log","@timestamp":"2021-02-24T18:28:36Z","tags":["error","elasticsearch","data"],"pid":1,"message":"[ConnectionError]: self signed certificate"}
│ {"type":"log","@timestamp":"2021-02-24T18:28:37Z","tags":["debug","metrics"],"pid":1,"message":"Refreshing metrics"}
│ {"type":"log","@timestamp":"2021-02-24T18:28:38Z","tags":["error","elasticsearch","data"],"pid":1,"message":"[ConnectionError]: self signed certificate"}
```

* When set `none`: Working!

```log
{"type":"log","@timestamp":"2021-02-24T18:32:58Z","tags":["debug","metrics"],"pid":1,"message":"Refreshing metrics"}
│ {"type":"ops","@timestamp":"2021-02-24T18:33:02Z","tags":[],"pid":1,"os":{"load":[3.65234375,2.638671875,2.5087890625],"mem":{"total":31562403840,"free":7950487552},"uptime":1758389},"
│ {"type":"log","@timestamp":"2021-02-24T18:33:03Z","tags":["debug","metrics"],"pid":1,"message":"Refreshing metrics"}
│ {"type":"ops","@timestamp":"2021-02-24T18:33:07Z","tags":[],"pid":1,"os":{"load":[3.52001953125,2.6279296875,2.505859375],"mem":{"total":31562403840,"free":7945019392},"uptime":1758394
│ {"type":"log","@timestamp":"2021-02-24T18:33:08Z","tags":["debug","metrics"],"pid":1,"message":"Refreshing metrics"}
│ {"type":"ops","@timestamp":"2021-02-24T18:33:12Z","tags":[],"pid":1,"os":{"load":[3.31787109375,2.6005859375,2.49755859375],"mem":{"total":31562403840,"free":7930966016},"uptime":17583
│ {"type":"log","@timestamp":"2021-02-24T18:33:13Z","tags":["debug","metrics"],"pid":1,"message":"Refreshing metrics"}
│ {"type":"ops","@timestamp":"2021-02-24T18:33:17Z","tags":[],"pid":1,"os":{"load":[3.37255859375,2.6240234375,2.505859375],"mem":{"total":31562403840,"free":7935848448},"uptime":1758404
│ {"type":"log","@timestamp":"2021-02-24T18:33:18Z","tags":["debug","metrics"],"pid":1,"message":"Refreshing metrics"}
│ {"type":"ops","@timestamp":"2021-02-24T18:33:22Z","tags":[],"pid":1,"os":{"load":[3.4228515625,2.64697265625,2.51416015625],"mem":{"total":31562403840,"free":7928467456},"uptime":17584
```
