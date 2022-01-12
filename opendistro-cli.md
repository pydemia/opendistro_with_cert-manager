# Open Distro CLI

CLI Downloads: https://opendistro.github.io/for-elasticsearch/
CLI Docs: https://opendistro.github.io/for-elasticsearch-docs/docs/cli/

## Installation

```
curl -fsSL https://d3g5vo6xdbdb9a.cloudfront.net/downloads/elasticsearch-clients/opendistro-cli/opendistro-odfe-cli-1.1.1-linux-x64.zip -o odfe-cli.zip && \
unzip ./odfe-cli.zip && \
chmod +x ./odfe-cli && \
chmod 600 ./odfe-cli && \
mkdir -p ~/.local/bin && \
cp ./odfe-cli ~/.local/bin/odfe-cli && \
odfe-cli --version
```

## Profile

### By Command

```bash
odfe-cli profile create  --name prd --auth-type basic --endpoint https://localhost:9200
```

### By `config.yaml`

```
mkdir -p ~/.odfe-cli && \
vim ~/.odfe-cli/config.yaml
```

```yaml
profiles:
    - name: prd
      endpoint: https://localhost:9200
      user: "admin"
      password: "foobar"
    - name: aws
      endpoint: https://some-cluster.us-east-1.es.amazonaws.com
      aws_iam:
        profile: ""
        service: es
```

## Usage

```bash
odfe-cli --profile prd curl get --path _cluster/settings
```
