# Open Distro for Elasticsearch build

This repo contains the scripts for building Open Distro for Elasticsearch Docker images in the directory docker/ and Linux distributions (RPM & DEB) in the directory linux_distributions/.

The default version to build is set in version.json for both Docker and Linux distributions.

## Getting started
```
git clone https://github.com/opendistro-for-elasticsearch/opendistro-build.git
```

Change directory to make sure you're in the right directory either elasticsearch/docker or elasticsearch/linux_distributions.

## Docker

### Credit

The docker build scripts are based on [elastic/elasticsearch-docker](https://github.com/elastic/elasticsearch-docker/tree/6.5)

The image is built on [CentOS 7](https://github.com/CentOS/sig-cloud-instance-images/blob/CentOS-7/docker/Dockerfile).

### Supported Docker versions

The images have been tested on Docker 18.09.2.

### Requirements

A full build and test requires the dependencies listed below.
The build scripts have been tested on Ubuntu 18.04 and Mac OS X. The following installation instructions assume Ubuntu 18.04.

- Docker
```
sudo apt-get update
sudo apt-get install docker.io
sudo usermod -a -G docker $USER
```
Then log out and back in again.

- GNU Make
- Python 3.5 with 

  pip
  ```
  sudo apt install python3-pip
  ```

  and virtualenv

  ```
  sudo apt-get install python-virtualenv
  ```

### Running a build

Make sure you are in the directory elasticsearch/docker.

Run
```
make build
```
to build an image based on the versions set in version.json

To build an image with a different version of Open Distro for Elasticsearch, either change version.json or run Make while specifying the exact version of Open Distro for Elasticsearch AND Elasticsearch.

For example:
```
OPENDISTRO_VERSION=0.8.0 ELASTIC_VERSION=6.6.2 make build
```

For running builds with a different repository name, run the following
```
OPENDISTRO_REPOSITORY=<repo_name> make build
```

To run a elasticsearch cluster with the docker images you just build, run the following
```
OPENDISTRO_VERSION=0.8.0 ELASTIC_VERSION=6.6.2 make run-cluster
OPENDISTRO_VERSION=0.8.0 ELASTIC_VERSION=6.6.2 make run-single
```

### Build a docker image with local artifacts

If you are interested in building a image with local artifacts, 
1. Download the artifacts under `build/elasticsearch/<new_folder>/elasticsearch-plugins` folder. 
2. Ensure that the relative path of the plugin zips to this folder is same as the ones mentioned in `PLUGIN_URL_PATHS`.
(e.g. the path for sql zip should be `build/elasticsearch/downloads/elasticsearch-plugins/opendistro-sql/opendistro-sql-0.7.0.0.zip` when `ARTIFACTS_DIR=downloads`. Any other path of the zip won't work. Other way is to override the plugin url path itself.)

```
ARTIFACTS_DIR=<new_folder> make
ARTIFACTS_DIR=<new_folder> PLUGIN_URL_PATHS=<opendistrozip path relative to <new_folder>/elasticsearch-plugins> make
```

### Testing the image
```
make test
```

## Linux Distributions

### Requirements

Make sure you are in the directory elasticsearch/linux_distributions folder and have Java installed (assume you are using Ubuntu 18.04).

- Java
```
sudo add-apt-repository ppa:openjdk-r/ppa
sudo apt-get update
sudo apt install openjdk-11-jdk
```

To build the rpm package
```
./gradlew buildRpm --console=plain -Dbuild.snapshot=false -b build.gradle
```

To build the deb package
```
./gradlew buildDeb --console=plain -Dbuild.snapshot=false -b build.gradle
```

To build rpm & deb packages
```
./gradlew buildPackages --console=plain -Dbuild.snapshot=false -b build.gradle
```

Download and install ES Linux Tar ball

The .tar.gz archive for Open distro for Elasticsearch v1.2.0 can be downloaded and installed as follows:
```
wget https://d3g5vo6xdbdb9a.cloudfront.net/tarball/opendistro-elasticsearch/opendistroforelasticsearch-1.2.0.tar.gz
wget https://d3g5vo6xdbdb9a.cloudfront.net/tarball/opendistro-elasticsearch/opendistroforelasticsearch-1.2.0.tar.gz.sha512 
shasum -a 512 -c opendistroforelasticsearch-1.2.0.tar.gz.sha512
tar -zxvf opendistroforelasticsearch-1.2.0.tar.gz
cd opendistroforelasticsearch-1.2.0
./opendistro-tar-install.sh
```
Single Node cluster:
```
./opendistro-tar-install.sh -Ecluster.name=my_cluster -Enode.name=node_1 -Ehttp.host=0.0.0.0 -Etransport.host=0.0.0.0 -Ediscovery.type=single-node
```


## Contributing

Open Distro for Elasticsearch is and will remain 100% open source under the Apache 2.0 license. As the project grows, we invite you to join the project and contribute. We want to make it easy for you to get started and remove friction — no lengthy Contributor License Agreement — so you can focus on writing great code.

## Questions

If you have any questions, please join our community forum [here](https://discuss.opendistrocommunity.dev/).

## Issues

File any issues [here](https://github.com/opendistro-for-elasticsearch/community/issues).


