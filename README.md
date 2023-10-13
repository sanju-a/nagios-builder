# nagios-builder
Docker build based compilation and RPM generation for nagios. This project also supports the multi-arch builds and works for x86 and arm. 

The Dockerfile has support for building the following components for nagios: 
    
    1. mk-livestatus from version 1.2.8p25 
    2. nagios-plugins version 2.4.6

Here are the commands required to build:

For ARM64
---------
$docker buildx build --progress=plain --platform=linux/arm64 --build-arg="ROCKY_LINUX_REL=9.2" --build-arg="NAGIOS_VERSION=4.4.13" --build-arg="BUILDPLATFORM=linux/arm64" --build-arg="TARGETPLATFORM=linux/arm64" --file dockerbuild/Dockerfile -t zreis-nagios-builder:4.4.13 .

For X86_64 / AMD64
-------------------
$docker buildx build --platform=linux/amd64 --build-arg "NAGIOS_VERSION=4.4.13" --build-arg "BUILDPLATFORM=linux/amd64" --build-arg "TARGETPLATFORM=linux/amd64" --file dockerbuild/Dockerfile -t zreis-nagios-builder:4.4.13 .

In order to copy the generated rpms and the mk-livestatus shared object (livestatus.so) and the object file (livestatus.o), you can either run the container from the built image or use the following command to get the rpms the required shared object and object files copied to /tmp on the host using the --output argument to docker.

For ARM64
-----------
$DOCKER_BUILDKIT=1 docker buildx build --progress=plain --platform=linux/arm64 --build-arg="ROCKY_LINUX_REL=9.2" --build-arg="NAGIOS_VERSION=4.4.13" --build-arg="BUILDPLATFORM=linux/arm64" --build-arg="TARGETPLATFORM=linux/arm64" --file dockerbuild/Dockerfile --output /tmp .

For x86_64/AMD64
-----------------
$DOCKER_BUILDKIT=1 docker buildx build --platform=linux/amd64 --build-arg "NAGIOS_VERSION=4.4.13" --build-arg "BUILDPLATFORM=linux/amd64" --build-arg "TARGETPLATFORM=linux/amd64" --file dockerbuild/Dockerfile --output /tmp .


