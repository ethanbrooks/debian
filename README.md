# Debian Base Image
[![Build Status](https://travis-ci.org/joeblackwaslike/debian.svg?branch=master)](https://travis-ci.org/joeblackwaslike/debian) [![Docker Pulls](https://img.shields.io/docker/pulls/joeblackwaslike/debian.svg)](https://hub.docker.com/r/joeblackwaslike/debian/) [![Github Repo](https://img.shields.io/badge/contributions-welcome-brightgreen.svg?style=flat)](https://github.com/joeblackwaslike/debian)

## Maintainer
Joe Black | <me@joeblack.nyc> | [github](https://github.com/joeblackwaslike)

## Introduction
The purpose of this project was to create an extremely minimal operating system for containers based on Debian.  Something comparable to the size of alpine containers but based on glibc, with bash included, and with the apt package manager.  The official debian images are currently 123MB in size.  This image on the other hand weighs in at only 59MB / 24MB compressed.

Many issues with running debian in a container were addressed as well.  This image only installs the bare minimum packages required to install a requested package, skipping anything reccomended or suggested by default.  This makes building a smaller image for services much simpler.  The included `apt-clean` command (courtesy of the upstream tklx/base project) is able to remove many things that apt-get clean misses (especially with the --aggressive option).


This image should work with all relatively recent debian releases, but those other than jessie and stretch are not tested.  To use a release other than jessie simply export the environment variable RELEASE to something other than `debian/jessie` before running building the rootfs.

This project makes extensive use of the tklx/base project (and internally chanko and debootstrap) and it would feel wrong to not mention it:  [tklx/base](https://github.com/tklx/base)

Pull requests with improvements always welcome.


## Featuring
* Everything that makes `tklx/base` amazing.
* Added the following packages to required:
    * netcat
    * procpcs and requirements: libncursesw5
    * bash-completion
    * iproute
    * libcap2-bin
* Numerous additional [Units](#units)
* Numerous additional [Scripts](#included-utilities-and-scripts)
* Numerous additional [Bash Functions](#included-bash-functions)


## Unit Changes
* Changes to `unit.d/apt.conf.d`:
    * [docker-autoremove-suggests](base-repo/unit.d/apt/overlay/etc/apt/apt.conf.d/docker-autoremove-suggests)
    * [no-install-suggests](base-repo/unit.d/apt/overlay/etc/apt/apt.conf.d/no-install-reccomends)

* Changes to `unit.d/etc-skel`
    * set .bashrc to `! shopt -q login_shell && . /etc/profile`


## Units
### `unit.d/etc-profile-d`:
Initialize an improved and functional shell environment and abstract complexity from entrypoint scripts and other helper scripts.

#### Config script
* Sets up dircolors

#### Overlay
**`/etc/profile.d/`:**
* [10-build-paths.sh](base-repo/unit.d/etc-profile-d/overlay/etc/profile.d/10-build-paths.sh)
* [30-shell-env.sh](base-repo/unit.d/etc-profile-d/overlay/etc/profile.d/30-shell-env.sh)
* [40-build-functions.sh](https://github.com/joeblackwaslike/container-utils/blob/master/functions/40-build-functions.sh)
* [40-useful-functions.sh](https://github.com/joeblackwaslike/container-utils/blob/master/functions/40-build-functions.sh)
* [50-kube-hostname-fix.sh](base-repo/unit.d/etc-profile-d/overlay/etc/profile.d/50-kube-hostname-fix.sh)
* [50-prompt-colors.sh](base-repo/unit.d/etc-profile-d/overlay/etc/profile.d/50-prompt-colors.sh)
* [60-build-environment.sh](base-repo/unit.d/etc-profile-d/overlay/etc/profile.d/60-build-environment.sh)


### `unit.d/util`:
A collection of helpers and utility scripts that help simplify and standardize common container tasks as well as work around issues that make running apps in containers difficult, or running particular apps in Kubernetes challenging.

#### Overlay
**`/`:**
* [dumb-init](https://github.com/Yelp/dumb-init)

**`/usr/local/bin/`:**
* [gosu](https://github.com/tianon/gosu)
* [goss](https://github.com/aelsabbahy/goss)
* [detect-proxy](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/detect-proxy)
* [erlang-cookie](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/erlang-cookie)
* [fixattrs](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/fixattrs)
* [kube-hostname-fix](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/kube-hostname-fix)
* [kube-search-domains](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/kube-search-domains)
* [path-helper](https://github.com/joeblackwaslike/joeblackwaslike/blob/master/scripts/path-helper)
* [set-limits](https://github.com/joeblackwaslike/joeblackwaslike/blob/master/scripts/set-limits)


### `unit.d/etc-environment-d`:
This creates a directory in `/etc/environment.d`. All files in this directory should be in the format `VARIABLE_NAME=values` but can also use bash parameter expansion as well as subshells to assign the values.  All files in this directory will be sourced in lexical order, then exported.

NOTE: *Please do not include exports in the files you you place under `/etc/environment.d`.*

NOTE: *You do not need to manually source `/etc/environment`, it will happen automatically in `/etc/profile.d`*


### `unit.d/etc-entrypoint-d`:
This creates a directory in `/etc/entrypoint.d`. All files in this directory are to be sourced at the top of the entrypoint script.

**Usage:**
```bash
. /etc/entrypoint
```


### `unit.d/etc-fixattrs-d`:
This creates a directory in `/etc/fixattrs.d`.  This directory is used to contain the fixattr files.

fixattrs allows you to write specification files during the build phase for ownerships and permissions to be set on container start.  Please see the `fixattrs` section under Included Scripts and Utilities.


### `unit.d/etc-paths-d`:
This creates a directory in `/etc/paths.d`.  This directory is used to contain files that contain path's to be added to `PATH`, one per line.

paths are only added to the path if they exist, they will be tested prior to adding them.  All paths are also deduplicated before being added.

For convenience, one entry already exists, which adds `~bin` if it exists.


### `unit.d/liboverridehostname`:
This installs `liboverridehostname.so.1` into `/usr/local/lib/`.  This library is used to override hostnames in containers without requiring any capabilities.  This opens up alot of possibilities for applications that have specific hostname requirements that can't be met using some cluster managers.

**Refs:**
* liboverridehostname: [github](https://github.com/joeblackwaslike/overridehostname)


## Included Utilities and Scripts

### `dumb-init`
https://github.com/Yelp/dumb-init

A simple and lightweight process supervisor that handles reaping zombies and properly passing signals.  It assumes PID1 and implements the expected interface of an application running as PID1.  Without dumb-init process supervision in containers is broken because the default signal handlers are disabled when running as PID1.  Without dumb-init your containers applications will not be able to shut down cleanly or properly.


### `gosu`
https://github.com/tianon/gosu

su and sudo have very strange TTY and signal forwarding behavior that break signaling as well as not passing things like PATH when preserving environment.

Overall it is unsuitable for container use, and gosu was created to address this specific problem and maintains parity with docker's own user implementation and code which it links against.

Using gosu to exec a process is much the same as having the container set to that `USER`. The version included has been compressed with UPX.

**Usage**:
```bash
exec gosu <user> <cmd>
```

### `kube-hostname-fix`
[/usr/local/bin/kube-hostname-fix](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/kube-hostname-fix)

This script works with `libfakehostname` and handles making sure the correct kubernetes pod hostname is reflected by `hostname` and makes use of `libfakehostname.so.1`.  is in /etc/hosts, exporting an updated `HOSTNAME`

After loading `libfakehostname.so.1` into `LD_PRELOAD`, you're able to use hostname normally in a container to set and get the container's hostname. This is accomplished by `libfakehostname.so.1` intercepting `gethostname(2)` and `sethostname(2)`.  Any application that uses these system calls will get the hostname you've set using `hostname`.

After setting the hostname, it ensures the FQDN is resolvable by editing the /etc/hosts file.  After that an updated `HOSTNAME` is exported as well.

**Usage**:
```bash
# enable hostname fix
eval $(kube-hostname-fix enable)

# disable's hostname fix
eval $(kube-hostname-fix disable)
```

NOTE: *This script is automatically activated through /etc/profile.d when `KUBE_HOSTNAME_FIX`=true, so in most cases it shouldn't be necessary to include in your entrypoint script.*

**Referencess:**
* liboverridehostname: [github](https://github.com/joeblackwaslike/overridehostname)


### `kube-search-domains`
[/usr/local/bin/kube-search-domains](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/kube-search-domains)

This script is used as a stop-gap measure until dns works correctly in kubernetes pods that are run with `hostNetwork: true`.  All it does is check whether the kubernetes search domains exist and if they do not, it adds them.


**NOTE:** In kubernetes 1.6+ you can set the `dnsPolicy: ClusterFirstWithHostNet` on your pod-spec and this script is not necessary at all anymore.


**Usage**:
```bash
# add
kube-search-domains add

# remove
kube-search-domains remove
```

NOTE: *This script is automatically activated through /etc/entrypoint.d when `KUBE_SEARCH_DOMAINS_ADD=true` or `KUBE_SEARCH_DOMAINS_REMOVE=true`, so in most cases it shouldn't be necessary to include in your entrypoint script.*


### `fixattrs`
[/usr/local/bin/fixattrs](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/fixattrs)

This script is a bash implementation of `fixattrs` from the s6 init project. Full usage @ [s6-documentation](https://github.com/just-containers/s6-overlay#fixing-ownership--permissions).  This allows you to write specification files during the build phase for ownerships and permissions to be set on container start.  It is necessary to include the `fixattrs` command in your entrypoint script because it would be much less useful to run it before the entrypoint script.  We suggest running it right before execing the app to be run.

**Usage**:
```bash
fixattrs
```


### `path-helper`
[/usr/local/bin/path-helper](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/path-helper)

Path helper allows you to write files to `/etc/paths.d` containing paths (one per line) to be added to the path of your environment.  This makes it much easier in the build script to add custom paths rather than trying to figure out the best place to add another path in the environment files.  All paths are read from these files, checked to make sure they exist, deduplicated, and added to the front of the existing `PATH`.


**Usage**:
```bash
# add a new path (during build)
echo '/some/path' >> /etc/paths.d/20-some-path
```

NOTE: *This script is automatically activated through /etc/profile.d, so in most cases it shouldn't be necessary to include in your entrypoint script.*


### `set-limits`
[/usr/local/bin/set-limits](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/set-limits)

This tool allows you to set ulimits for a container in the convenient declarative syntax of limits config files.

Be sure to copy your *.limits.conf files in `/etc/security/limits.d` named as: `<name>.limits.conf`.

You will need to make sure you add the necessary capabilities such as `SYS_RESOURCE` and `SYS_NICE` to your container, otherwise in many cases, you will not be able to raise certain limits.

**Usage**:
```bash
# to use /etc/security/limits.d/test.limits.conf
set-limits test
```


### `detect-proxy`
[/usr/local/bin/detect-proxy](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/detect-proxy)

Used when building a container locally, auto detects a local proxy running on the bridgeip address and/or apt-cacher proxy and sets up `http_proxy` and Apt appropriately.

Proxy detection tests the bridge ip for ports 3142 for apt-cacher and 3128 for squid or other generic http proxy.

**Usage**:
```bash
# at beginning of build script
eval $(detect-proxy enable)

# at end of build script
eval $(detect-proxy disable)
```


### `erlang-cookie`
[/usr/local/bin/erlang-cookie](https://github.com/joeblackwaslike/container-utils/blob/master/scripts/erlang-cookie)

Writes the value of `ERLANG_COOKIE || insecure-cookie` to `~/.erlang-cookie`.

**Usage**:
```bash
erlang-cookie write
```

NOTE: *This script is automatically activated through /etc/profile.d when `ERLANG_COOKIE` is present, so in most cases it shouldn't be necessary to include in your entrypoint script.*


## Included Bash Functions
NOTE: *All of these functions are in the global environment so should be usable anywhere and in scripts that begin with `#!/bin/bash -l`. For this reason many are namespaced to prevent collisions.*

### build time
[/etc/profile.d/40-build-functions.sh](https://github.com/joeblackwaslike/container-utils/blob/master/functions/40-build-functions.sh)
* `build::apt::get-version`
* `build::apt::add-key`
* `build::user::create`

### run time / entrypoint use
[/etc/profile.d/40-useful-functions.sh](https://github.com/joeblackwaslike/container-utils/blob/master/functions/40-useful-functions.sh)
* `clear`
* `container-id`
* `get-ipv4`
* `log::get-time`
* `log::info`
* `log::error`
* `log::warn`
* `log::m-info`
* `log::m-warn`
* `log::m-error`
* `linux::cmd::exists`
* `linux::cmd::does-not-exist`
* `linux::cap::is-enabled`
* `linux::cap::is-disabled`
* `linux::cap::show-warning`
* `linux::get-shell-vars`
* `linux::get-shell-var-names`
* `linux::get-env-vars`
* `linux::get-env-var-names`
* `linux::get-function-names`
* `net::is-long-hostname`
* `net::is-short-hostname`
* `net::get-mtu`
* `kube::is-kubernetes`
* `kube::dns::get-cluster-domain`
* `kube::sd::get-port`
* `kube::sd::get-hosts`
* `kube::sd::num-hosts`
* `kube::api::pod::ip-to-hostname`
* `kube::api::endpoint::get-hosts`
* `kube::api::endpoint::get-ips`
* `kube::api::endpoint::get-nodes`
* `kube::api::endpoint::get-ports`
* `kube::api::endpoint::get-port`
* `kube::api::service::get`
* `kube::api::endpoint::get`
* `kube::api::pod::get`
* `kube::api::statefulset::get`
* `kube::api::deployment::get`
* `kube::api::replicaset::get`
* `kube::api::daemonset::get`
* `kube::api::configmap::get`
* `kube::api::secret::get`
* `kube::api::namespace::get`
* `kube::api::ingress::get`
* `kube::api::node::get`
* `kube::api::pv::get`
* `kube::api::pvc::get`
* `kube::api::job::get`
* `kube::api::event::get`
* `kube::api::networkpolicy::get`
* `kube::api::is-namespaced`
* `kube::api::is-not-namespaced`
* `kube::sa::get-namespace`
* `kube::sa::get-token`
* `kube::resolv::get-domain`
* `kube::host::is-statefulset`
* `kube::host::isnt-statefulset`
* `kube::host::get-domain`
* `kube::host::get-namespace`
* `kube::host::get-service`
* `kube::host::get-node`
* `kube::host::get-statefulset`
* `kube::host::get-index`
* `kube::host::is-master`
* `kube::host::get-hostname`
* `kube::host::get-subdomain`
* `erlang::vmargs::get-name`
* `erlang::set-erl-dump`
* `kazoo::sup`
* `kazoo::release::get-version`
* `kazoo::erts::get-version`
* `kazoo::build-amqp-uri`
* `kazoo::build-amqp-uris`
* `kazoo::build-amqp-uri-list`
* `shell::is-interactive`
* `shell::is-not-interactive`


## Using this image
You can use this image as a base image in your dockerfile:
```
FROM joeblackwaslike/debian:jessie

FROM joeblackwaslike/debian:stretch
```

Or as a lightweight disposable container for testing:
```bash
docker run -it --rm joeblackwaslike/debian:jessie bash
```

See [telephone repositories](https://github.com/telephoneorg) for numerous images that use this image as their base.


## Gotchas

### Problem:
I ran `apt-get install curl` and curl isn't working for sites with TLS.

### Answer:
This minimal distribution has some changes made to `apt.conf.d` that alter apt-get's behavior.  Suggested and recommended packages are no longer installed by default, and closely equivalent to running `apt-get install --no-install-recommends <pkg>`.  Therefore you need to be more explicit if what you are installing depends on packages that are only recommended or suggested.  In the case of curl, you can fix this by issuing `apt-get install -y curl ca-certificates` in order to pull in the root ca's and also setup requirements such as gnutls.
