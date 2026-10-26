# ProtonMail IMAP/SMTP Bridge Docker Container

This is an unofficial Docker container of the [ProtonMail Bridge](https://protonmail.com/bridge/). Some of the scripts are based on [Hendrik Meyer's work](https://gitlab.com/T4cC0re/protonmail-bridge-docker).

This repository is a fork of [shenxn/protonmail-bridge-docker](https://github.com/shenxn/protonmail-bridge-docker), maintained at [drygoatair/protonmail-bridge-docker](https://github.com/drygoatair/protonmail-bridge-docker).

Docker Hub (upstream images, not published from this fork): [https://hub.docker.com/r/shenxn/protonmail-bridge](https://hub.docker.com/r/shenxn/protonmail-bridge)

### Differences from upstream

 - The build image installs `libfido2-dev` and `libcbor-dev` (and `libfido2-1` at runtime), which recent Bridge versions need to compile.
 - The example [docker-compose.yml](docker-compose.yml) builds the image locally from `./build` and attaches the service to a named `protonmail-network` network.
 - The daily update check tracks new upstream Bridge releases in `VERSION`.

## ARM Support

We now support ARM devices (`arm64` and `arm/v7`)! Use the images tagged with `build`. See next section for details.

## Tags

There are two types of images.
 - `deb`: Images based on the official [.deb release](https://protonmail.com/bridge/install). It only supports the `amd64` architecture.
 - `build`: Images based on the [source code](https://github.com/ProtonMail/proton-bridge). It supports `amd64`, `arm64`, `arm/v7` and `riscv64`. Supporting to more architectures is possible. PRs are welcome.

tag | description
 -- | --
`latest` | latest `deb` image
`[version]` | `deb` images
`build` | latest `build` image
`[version]-build` | `build` images

## Initialization

To initialize, create a copy of the provided example [docker-compose.yml](docker-compose.yml) file, modify it to suit your needs, and then run the following command:

```
docker compose run protonmail-bridge init
```

Wait for the bridge to startup, then you will see a prompt appear for [Proton Mail Bridge interactive shell](https://proton.me/support/bridge-cli-guide). Use the `login` command and follow the instructions to add your account into the bridge. Then use `info` to see the configuration information (username and password). After that, use `exit` to exit the bridge. You may need `CTRL+C` to exit the docker entirely.

## Run

To run the container, use the following command.

```
docker run -d --name=protonmail-bridge -v protonmail:/root -p 1025:25/tcp -p 1143:143/tcp --restart=unless-stopped shenxn/protonmail-bridge
```

Or, if using Docker Compose, use the following command.

```
docker compose up -d
```

## Kubernetes

If you want to run this image in a Kubernetes environment. You can use the [Helm](https://helm.sh/) chart (https://github.com/k8s-at-home/charts/tree/master/charts/stable/protonmail-bridge) created by [@Eagleman7](https://github.com/Eagleman7). More details can be found in [#23](https://github.com/shenxn/protonmail-bridge-docker/issues/23).

If you don't want to use Helm, you can also reference to the guide ([#6](https://github.com/shenxn/protonmail-bridge-docker/issues/6)) written by [@ghudgins](https://github.com/ghudgins).

## Security

Please be aware that running the command above will expose your bridge to the network. Remember to use firewall if you are going to run this in an untrusted network or on a machine that has public IP address. You can also use the following command to publish the port to only localhost, which is the same behavior as the official bridge package.

```
docker run -d --name=protonmail-bridge -v protonmail:/root -p 127.0.0.1:1025:25/tcp -p 127.0.0.1:1143:143/tcp --restart=unless-stopped shenxn/protonmail-bridge
```

Besides, you can publish only port 25 (SMTP) if you don't need to receive any email (e.g. as a email notification service).

## Compatibility

The bridge currently only supports some of the email clients. More details can be found on the official website. I've tested this on a Synology DiskStation and it runs well. However, you may need ssh onto it to run the interactive docker command to add your account. The main reason of using this instead of environment variables is that it seems to be the best way to support two-factor authentication.

## Bridge CLI Guide

The initialization step exposes the bridge CLI so you can do things like switch between combined and split mode, change proxy, etc. The [official guide](https://protonmail.com/support/knowledge-base/bridge-cli-guide/) gives more information on to use the CLI.

## Build

For anyone who want to build this container on your own (for development or security concerns), here is the guide to do so. First, you need to `cd` into the directory (`deb` or `build`, depending on which type of image you want). Then just run the docker build command
```
docker build --build-arg version=$(cat ../VERSION) .
```

The `build` image requires the `version` build argument (the upstream Bridge git tag, e.g. `v3.27.1`). The `deb` image does not need it. Note that `docker compose build` passes no `version`, so the clone falls back to the upstream default branch rather than the release in `VERSION`; add `args: {version: v3.27.1}` under `build:` in your compose file to pin a release.

That's it. The `Dockerfile` and bash scripts handle all the downloading, building, and packing. You can also add tags, push to your favorite docker registry, or use `buildx` to build multi architecture images.
