[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

![piku logo](./img/logo.png)

The tiniest Heroku/CloudFoundry-like PaaS you've ever seen. **Seldom updated because it is _stable_** and used in production daily by several people.

`piku`, inspired by [dokku][dokku], allows you do `git push` deployments to your own servers.

[![asciicast](https://asciinema.org/a/Ar31IoTkzsZmWWvlJll6p7haS.svg)](https://asciinema.org/a/Ar31IoTkzsZmWWvlJll6p7haS)

### Documentation: [Using](#using-piku) | [Install](#install) | [Procfile](docs/DESIGN.md#procfile-format) | [ENV](./docs/ENV.md) | [Examples](./examples/README.md) | [Roadmap](https://github.com/piku/piku/projects/2) | [Contributing](./docs/CONTRIBUTING.md) | [LinuxConf Talk](https://www.youtube.com/watch?v=ec-GoDukHWk) | [Fast Web App Tutorial](https://github.com/piku/webapp-tutorial)

## Project Activity / Deprecation Notices

**`piku` is considered STABLE**. It is actively maintained, but "actively" here means the feature set is pretty much done, so it is only updated when new runtimes are added or reproducible bugs crop up.

It currently requires Python 3.5 or above, but will move to require 3.8+ sometime in late 2022 since that was the baseline Python 3 version in Ubuntu LTS 20.04 and Debian 11 has already moved on to 3.9. Since most of its users run it on LTS distributions, there is no rush to introduce disruption. The current plan is to throw up a warning for older runtimes and do regression testing for 3.7, 3.8, 3.9 and 3.10 (replacing the current bracket of tests from 3.5 to 3.8), and make sure we also cover Ubuntu 22.04, Debian 11 and Fedora 36+.

## Goals and Motivation

I kept finding myself wanting an Heroku/CloudFoundry-like way to deploy stuff on a few remote ARM boards and [my Raspberry Pi cluster][raspi-cluster], but since [dokku][dokku] didn't work on ARM at the time and even `docker` can be overkill sometimes, I decided to roll my own.

### Core values

 * Runs on low end devices.
 * Accessible to hobbyists and K-12 schools.
 * ~1000 lines readable code.
 * Functional code style.
 * Few (single?) dependencies
 * [12 factor app](https://12factor.net).
 * Simplify user experience.
 * Cover 80% of common use cases.
 * Sensible defaults.
 * Leverage distro packages in Raspbian/Debian/Ubuntu (Alpine and RHEL support is WIP)
 * Leverage standard tooling (`git`, `ssh`, `uwsgi`, `nginx`).
 * Preserve backwards compatibility where possible

## Using `piku`

`piku` supports a Heroku-like workflow, like so:

* Create a `git` SSH remote pointing to your `piku` server with the app name as repo name.
  `git remote add piku piku@yourserver:appname`.
* Push your code: `git push piku master` (or if you want to push a different branch than the current one use `git push piku release-branch-name`).
* `piku` determines the runtime and installs the dependencies for your app (building whatever's required).
   * For Python, it segregates each app's dependencies into a `virtualenv`.
   * For Go, it defines a separate `GOPATH` for each app.
   * For Node, it installs whatever is in `package.json` into `node_modules`.
   * For Java, it builds your app depending on either `pom.xml` or `build.gradle` file.
   * For Ruby, it does `bundle install` of your gems in an isolated folder.
* It then looks at a [`Procfile` which is documented here](docs/DESIGN.md#procfile-format) and starts the relevant workers using [uWSGI][uwsgi] as a generic process manager.
* You can optionally also specify a `release` worker which is run once when the app is deployed.
* You can then remotely change application settings (`config:set`) or scale up/down worker processes (`ps:scale`).
* You can also bake application settings into a file called [`ENV` which is documented here](./docs/ENV.md).
* A `static` worker type, with the root path as the argument, can be used to deploy a gh-pages style static site.

## Install

To use `piku` you need a VPS, Raspberry Pi, or other server bootstrapped with `piku`'s requirements. You can use a single server to run multiple `piku` apps.

There are two main ways of deploying `piku` onto a new server:

* Use [`piku-bootstrap`](https://github.com/piku/piku-bootstrap) to reconfigure a new or existing Ubuntu virtual machine
* Use `cloud-init` when creating a new virtual machine or barebones automated deployment (check [this repository](https://github.com/piku/cloud-init) for examples)

### `piku` client

To make life easier you can also install the [piku](./piku) helper CLI. Install it into your path e.g. `~/bin` to run it from anywhere.

```shell
curl https://raw.githubusercontent.com/piku/piku/master/piku > ~/bin/piku && chmod 755 ~/bin/piku
```

This shell script makes working with `piku` remotes a bit simpler. If you have a git remote called `piku` in the current folder it will infer the remote server and app name and insert those into the remote piku commands. This allows you to execute commands like the following on your running remote app:

```shell
$ piku logs
$ piku config:set MYVAR=12
$ piku stop
$ piku deploy
$ piku destroy
$ piku # <- will show help for the remote app
```

Run `piku` on it's own to see the available remote and local commands.

You can use the `init` command to download an example Procfile and ENV file into the current folder:

```shell
$ piku init
Wrote ./ENV file.
Wrote ./Procfile.
```

You can pass flags through to the underlying SSH command, for example `-t` to run interactive commands remotely, and `-A` to proxy authentication credentials in order to do remote git pulls.

Here is an example of using the `-t` flag to obtain a `bash` shell in the app directory of one of your Piku apps:

```shell
$ piku -t run bash
Piku remote operator.
Server: piku@cloud.mccormickit.com
App: dashboard

piku@piku:~/.piku/apps/dashboard$ ls
data  ENV  index.html  package.json  package-lock.json  Procfile  server.wisp
```

Tip: If you put this `piku` script on your `PATH` you can use the `piku` command across multiple apps on your local.

## Virtual Hosts

If you are on a LAN and are accessing `piku` from macOS/iOS/Linux clients, you can try using [`piku/avahi-aliases`](https://github.com/piku/avahi-aliases) to announce different hosts via Avahi/mDNS/Bonjour.

## Supported Platforms

`piku` is intended to work in any POSIX-like environment where you have Python, [uWSGI][uwsgi] and SSH, i.e.: 
Linux, FreeBSD, [Cygwin][cygwin] and the [Windows Subsystem for Linux][wsl].

As a baseline, it began its development on an original, 256MB Rasbperry Pi Model B, and still runs reliably on it.

Since I have an ODROID-U2, [a bunch of Pi 2s][raspi-cluster] and a few more ARM boards on the way, it is often tested on a number of places where running `x64` binaries is unfeasible.

But there are already a few folk using `piku` on vanilla `x64` Linux without any issues whatsoever, so yes, you can use it as a micro-PaaS for 'real' stuff. Your mileage may vary.

## Supported Runtimes

`piku` currently supports deploying apps (and dependencies) written in Python, with Go, Clojure (Java) and Node (see [above](#project-statustodo)) in the works. But if it can be invoked from a shell, it can be run inside `piku`.

[click]: http://click.pocoo.org
[pi]: http://www.raspberrypi.org
[dokku]: https://github.com/dokku/dokku
[raspi-cluster]: https://github.com/rcarmo/raspi-cluster
[cygwin]: http://www.cygwin.com
[uwsgi]: https://github.com/unbit/uwsgi
[wsl]: https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux
