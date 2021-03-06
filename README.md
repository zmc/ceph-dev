# RHCS Dashboard Dev. Env.

## Installation

* If it doesn't exist, create a local directory for **ccache**:
```
mkdir -p ~/.ccache
```

* Clone Ceph:
```
git clone git@github.com:ceph/ceph.git
```

* Clone rhcs-dashboard/ceph-dev:
```
git clone git@github.com:rhcs-dashboard/ceph-dev.git
```

* In ceph-dev, create *.env* file from template and set values:
```
cd ceph-dev
cp .env.example .env

# default values:

HOST_CCACHE_DIR=/path/to/your/local/.ccache/dir

CEPH_IMAGE_TAG=fedora29
CEPH_REPO_DIR=/path/to/your/local/ceph/repo
# Optional: a custom build directory other than default one ($CEPH_REPO_DIR/build)
CEPH_CUSTOM_BUILD_DIR=
# Set 5200 if you want to access the dashboard proxy at http://localhost:5200
CEPH_PROXY_HOST_PORT=4200
# Set 11001 if you want to access the dashboard at https://localhost:11001
CEPH_HOST_PORT=11000

GRAFANA_HOST_PORT=3000
PROMETHEUS_HOST_PORT=9090
NODE_EXPORTER_HOST_PORT=9100
ALERTMANAGER_HOST_PORT=9093
```

* Install [Docker Compose](https://docs.docker.com/compose/install/). If your OS is Fedora/CentOS/RHEL, you can run:
```
# Fedora:
sudo bash ./scripts/docker/install-docker-compose-fedora.sh

# CentOS/RHEL:
sudo bash ./scripts/docker/install-docker-compose-centos-rhel.sh
```

If you ran the above script, then you can run *docker* and *docker-compose* without *sudo* if you log out and log in.

* Download docker images:
```
docker-compose pull
```

* Optionally, set up git pre-commit hook:
```
docker-compose run --rm -e HOST_PWD=$PWD ceph /docker/ci/pre-commit-setup.sh
```

## Usage

* Build Ceph (with python 3: CEPH_IMAGE_TAG=fedora29; with python 2: CEPH_IMAGE_TAG=centos7):
```
docker-compose run --rm ceph /docker/build-ceph.sh
```

* Start ceph + dashboard feature services:
```
docker-compose up -d

# Only ceph:
docker-compose up -d ceph
```

* Display ceph container logs:
```
docker-compose logs -f ceph
```

* Access dashboard with credentials: admin / admin

https://localhost:$CEPH_HOST_PORT

Access dev. server dashboard when you see in container logs something like this:
```
ceph    | ℹ ｢wdm｣: Compiled successfully.
```

http://localhost:$CEPH_PROXY_HOST_PORT

* Restart dashboard module:
```
docker-compose exec ceph /docker/restart-dashboard.sh
```

* Add OSD in 2nd host (once ceph dashboard is accessible):
```
docker-compose up -d --scale ceph-host2=1
```

* Stop all:
```
docker-compose down
```

* Rebuild not proxied dashboard frontend:
```
docker-compose run --rm ceph /docker/build-dashboard-frontend.sh
```

* Run backend unit tests:
```
docker-compose run --rm ceph /docker/ci/run-backend-unit-tests.sh

# Only specific tests:
docker-compose run --rm ceph /docker/ci/run-backend-unit-tests.sh tests/test_rest_client.py tests/test_grafana.py

# Only 1 test:
docker-compose run --rm ceph /docker/ci/run-backend-unit-tests.sh tests/test_rgw_client.py::RgwClientTest::test_ssl_verify
```

* Run API tests (based on Teuthology: Ceph integration test framework):
```
docker-compose run --rm ceph /docker/ci/run-api-tests.sh

# Only specific tests:
docker-compose run --rm ceph /docker/ci/run-api-tests.sh tasks.mgr.dashboard.test_health tasks.mgr.dashboard.test_pool
```

* Run frontend E2E tests:
```
# If ceph is running:
docker-compose exec ceph /docker/e2e/run-frontend-e2e-tests.sh

# If ceph is not running:
docker-compose run --rm ceph /docker/e2e/run-frontend-e2e-tests.sh
```

* Run sanity checks:
```
docker-compose run --rm ceph /docker/ci/run-sanity-checks.sh
```

* Build Ceph documentation:
```
docker-compose run --rm ceph /docker/build-doc.sh
```

* Display Ceph documentation:
```
docker-compose run --rm -p 11001:8080 ceph admin/serve-doc

# Access here: http://localhost:11001
```

## Grafana

If you have started grafana, you can access it at:

http://localhost:$GRAFANA_HOST_PORT/login

## Prometheus

If you have started prometheus, you can access it at:

http://localhost:$PROMETHEUS_HOST_PORT

## Single Sign-On (SSO)

Keycloak (open source Identity and Access Management solution)
will be used for authentication, so localhost port 8080 must be available.

* Start Keycloak:
```
docker-compose up -d --scale keycloak=1 keycloak
```

You can access Keycloak administration console with credentials: admin / keycloak

http://localhost:8080

* Enable dashboard SSO in running ceph container:
```
docker-compose exec ceph /docker/sso/sso-enable.sh
```

* Access dashboard with SSO credentials: admin / ssoadmin

https://localhost:$CEPH_HOST_PORT

* Disable dashboard SSO:
```
docker-compose exec ceph /docker/sso/sso-disable.sh
```

## Build and push an image to docker registry:

If you want to update an image, you'll have to edit image's Dockerfile and then:

* Build image:
```
docker build -t {imageName}:{imageTag} -f {/path/to/Dockerfile} ./docker/ceph

# Example:
docker build -t rhcsdashboard/ceph:fedora29 -f ./docker/ceph/fedora/Dockerfile ./docker/ceph
docker build -t rhcsdashboard/ceph:centos7 -f ./docker/ceph/centos/Dockerfile  ./docker/ceph
```

* Optionally, create an additional tag:
```
docker tag {imageName}:{imageTag} {imageName}:{imageNewTag}

# Example:
docker tag rhcsdashboard/ceph:fedora29 rhcsdashboard/ceph:latest
docker tag rhcsdashboard/ceph:centos7 rhcsdashboard/ceph:latest
```

* Log in to rhcs-dashboard docker registry:
```
docker login -u rhcsdashboard
```

* Push image:
```
docker push {imageName}:{imageTag}

# Example:
docker push rhcsdashboard/ceph:fedora29
docker push rhcsdashboard/ceph:centos7
```

## Start Ceph 2 (useful for parallel development)

* Set appropriate values in *.env*:
```
CEPH2_REPO_DIR=/path/to/your/local/ceph2
CEPH2_CUSTOM_BUILD_DIR=
# default: 4202
CEPH2_PROXY_HOST_PORT=4202
# default: 11002
CEPH2_HOST_PORT=11002
```

* Start ceph2 + ceph + ...:
```
docker-compose up -d --scale ceph2=1

# Start ceph2 but not ceph:
docker-compose up -d --scale ceph2=1 --scale ceph=0
```

## Start RHCS 3.2 RPM version

* Set appropriate values in *.env* (you have to be logged into private registry):
```
RHCS3_2_IMAGE=docker-registry.engineering.redhat.com/ceph-dashboard/rhcs3.2
# default: 11032
RHCS3_2_HOST_PORT=11032
```

* Start rhcs3.2 + ceph + ...:
```
docker-compose up -d --scale rhcs3.2=1

# Start rhcs3.2 but not ceph:
docker-compose up -d --scale rhcs3.2=1 --scale ceph=0
```
