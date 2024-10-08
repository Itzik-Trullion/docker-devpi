docker-devpi
============

This repository contains a Dockerfile for [devpi pypi server](http://doc.devpi.net/latest/).

You can use this container to speed up the `pip install` parts of your docker
builds. This is done by adding an optional cache of your requirement python
packages and speed up docker. The outcome is faster development without
breaking builds.

## Getting started

### Installation

```bash
docker pull dgunchev/devpi
```

### Quickstart

Start using

```bash
docker run -d --name devpi \
    --publish 3141:3141 \
    --volume /srv/docker/devpi:/data \
    --env=DEVPI_PASSWORD=changemetoyourlongsecret \
    --restart always \
    dgunchev/devpi
```

*Alternatively, you can use the sample [docker-compose.yml](docker-compose.yml)
file to start the container using [Docker Compose](https://docs.docker.com/compose/)*

Please set ``DEVPI_PASSWORD`` to a secret otherwise an attacker can *execute
arbitrary code*.

### Client side usage

To use this devpi cache to speed up your dockerfile builds, add the code below
in your dockerfiles. This will add the devpi container an optional cache for
pip. The docker containers will try using port 3141 on the docker host first
and fall back on the normal pypi servers without breaking the build.

#### For Debian/Ubuntu/deb based images

```Dockerfile
# Install iproute2
RUN apt-get update \
 && apt-get install -y iproute2 \
 && rm -rf /var/lib/apt/lists/*

 # Use an optional pip cache to speed development
RUN export HOST_IP=$(ip route| awk '/^default/ {print $3}') \
 && mkdir -p ~/.pip \
 && echo [global] >> ~/.pip/pip.conf \
 && echo extra-index-url = http://$HOST_IP:3141/app/dev/+simple >> ~/.pip/pip.conf \
 && echo [install] >> ~/.pip/pip.conf \
 && echo trusted-host = $HOST_IP >> ~/.pip/pip.conf \
 && cat ~/.pip/pip.conf
```

#### For new Fedora/Redhat/rpm+dnf based images

```Dockerfile
# Install iproute
RUN dnf install iproute \
 && dnf clean all

 # Use an optional pip cache to speed development
RUN export HOST_IP=$(ip route| awk '/^default/ {print $3}') \
 && mkdir -p ~/.pip \
 && echo [global] >> ~/.pip/pip.conf \
 && echo extra-index-url = http://$HOST_IP:3141/app/dev/+simple >> ~/.pip/pip.conf \
 && echo [install] >> ~/.pip/pip.conf \
 && echo trusted-host = $HOST_IP >> ~/.pip/pip.conf \
 && cat ~/.pip/pip.conf
```

#### For old Fedora/Redhat/rpm+yum based images

```Dockerfile
# Install iproute
RUN yum install iproute \
 && yum clean all

 # Use an optional pip cache to speed development
RUN export HOST_IP=$(ip route| awk '/^default/ {print $3}') \
 && mkdir -p ~/.pip \
 && echo [global] >> ~/.pip/pip.conf \
 && echo extra-index-url = http://$HOST_IP:3141/app/dev/+simple >> ~/.pip/pip.conf \
 && echo [install] >> ~/.pip/pip.conf \
 && echo trusted-host = $HOST_IP >> ~/.pip/pip.conf \
 && cat ~/.pip/pip.conf
```

### The default IP can be found using python too:

```bash
python3 -c 'import socket; sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM); \
    sock.connect(("8.8.8.8", 1)); print(sock.getsockname()[0])'
```

### Uploading python packages files

You need to upload your python requirement to get any benefit from the devpi
container. You can upload them using the bash code below a similar build
environment.

```bash
pip wheel --download=packages --wheel-dir=wheelhouse -r requirements.txt
pip install "devpi-client>=2.3.0" \
&& export HOST_IP=$(ip route| awk '/^default/ {print $3}') \
&& if devpi use http://$HOST_IP:3141>/dev/null; then \
       devpi use http://$HOST_IP:3141/root/public --set-cfg \
    && devpi login root --password=$DEVPI_PASSWORD  \
    && devpi upload --from-dir --formats=* ./wheelhouse ./packages; \
else \
    echo "No started devpi container found at http://$HOST_IP:3141"; \
fi
```

## Persistence

For devpi to preserve its state across container shutdown and startup, you
should mount a volume at `/data`. The quickstart command already includes this.

## Security

Devpi creates a user named root by default.
Its password should be set with the `DEVPI_PASSWORD` environment variable.
Please set it, otherwise attackers can *execute arbitrary code* in your application
by uploading modified packages.

For additional security the argument `--restrict-modify root` has been added, so
only the root may create users and indexes.

## History

This is a fork the [Centre for Comparative Genomics, Murdoch
University](https://github.com/muccg)'s [docker-devpi](https://github.com/muccg/docker-devpi)
repository, which was last updated in 2018-05-24 (2.5 years old at the time).
I needed something fresher so I forked it.

I managed to get the image size down to 29% of the original size. Initially
it used the python:3.9.1 image and resulted in 994 MB image. Switching to
fedora:latest resulted in 493 MB. Adding `dnf clean all` the image is
now down to 282 MB (or ~ 90 MB compressed).

Building with alpine linux got the image down to 135 MB, 7.3 times smaller.
The compressed image is 54.45 MB. This was a good exercise - building in
one container with gcc, copying the compiled files and installing them in another
container without gcc... cool. Also removed 1.9 MB of `APKINDEX` gzip caches.
