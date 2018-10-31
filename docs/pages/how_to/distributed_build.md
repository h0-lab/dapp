---
title: Distributed build
sidebar: how_to
permalink: how_to/distributed_build.html
author: Artem Kladov <artem.kladov@flant.com>
---

## Task Overview

Dapp can store stages cache in a Docker registry. It gives an opportunity to use a distributed build in the following cases:
* **Dynamic resources.** You can setup a dynamic allocation of resources for building by using cloud providers. These resources can be temporary so in time of build, a stages cache may be absent.
* **More than one build nodes.** If you have more than one nodes (hosts) for building images, you need to synchronize stages cache before building an image.

Distributed build assumes the following steps:
* pull existing stages cache from the Docker registry;
* build images;
* push built images and stages cache to the Docker registry.

In this tutorial, we will build an image of simple PHP [Symfony application](https://github.com/symfony/demo) using distributed build on two build nodes.

## Requirements

Two hosts with dapp installed.

> You can use one host instead of two, but in this case, you need to clean stages cache by executing `dapp dimg stages flush local` command before pulling stages cache.

## Building the application on the node1

Make the following steps on `node1`.

### Building the application

Clone the [Symfony Demo Application](https://github.com/symfony/demo) repository to get the source code:

```shell
git clone https://github.com/symfony/symfony-demo.git
```

In the project root directory create a `dappfile.yaml` with the following content:

{% raw %}
```yaml
dimg: ~
from: ubuntu:16.04
docker:
  WORKDIR: /app
  # Non-root user
  USER: app
  EXPOSE: "80"
  ENV:
    LC_ALL: en_US.UTF-8
ansible:
  beforeInstall:
  - name: "Install additional packages"
    apt:
      name: "{{`{{ item }}`}}"
      state: present
      update_cache: yes
    with_items:
      - locales
      - ca-certificates
  - name: "Generate en_US.UTF-8 default locale"
    locale_gen:
      name: en_US.UTF-8
      state: present
  - name: "Create non-root group for the main application"
    group:
      name: app
      state: present
      gid: 242
  - name: "Create non-root user for the main application"
    user:
      name: app
      comment: "Create non-root user for the main application"
      uid: 242
      group: app
      shell: /bin/bash
      home: /app
  - name: Add repository key
    apt_key:
      keyserver: keyserver.ubuntu.com
      id: E5267A6C
  - name: "Add PHP apt repository"
    apt_repository:
      repo: 'deb http://ppa.launchpad.net/ondrej/php/ubuntu xenial main'
      update_cache: yes
  - name: "Install PHP and modules"
    apt:
      name: "{{`{{ item }}`}}"
      state: present
      update_cache: yes
    with_items:
      - php7.2
      - php-sqlite3
      - php-xml
      - php-zip
      - php-mbstring
      - php-intl
  - name: Install composer
    get_url:
      url: https://getcomposer.org/download/1.6.5/composer.phar
      dest: /usr/local/bin/composer
      mode: a+x
  install:
  - name: "Install app deps"
    # NOTICE: Always use `composer install` command in real world environment!
    shell: composer update
    become: yes
    become_user: app
    args:
      creates: /app/vendor/
      chdir: /app/
  setup:
  - name: "Create start script"
    copy:
      content: |
        #!/bin/bash
        php bin/console server:run 0.0.0.0:8000
      dest: /app/start.sh
      owner: app
      group: app
      mode: 0755
  - raw: echo `date` > /app/version.txt
  - raw: chown app:app /app/version.txt
git:
- add: /
  to: /app
  owner: app
  group: app
```   
{% endraw %}

From the project root directory (as and forth for all dapp commands) execute the following command:

```shell
dapp dimg build
```

### Pushing stages cache

Login to your Docker registry:

```shell
docker login <registry_address>
```
> If you don't have a Docker registry, you can register on the [GitLab](https://gitlab.com) and use registry there.

Push built image and stages cache:

```shell
dapp dimg push --with-stages <registry_address>
```

## Building the application on the node2 using saved stages cache

In your remote Docker registry there are several images, named like `dimgstage-bda2d7aef23fafa50e29e2a00812dad14f2249e5b123598f8911555717a9160e`. There are images of the  stages cache.

First of all, make the following steps on `node2`:

* Clone the [Symfony Demo Application](https://github.com/symfony/demo) repository as you made in on the corresponding step earlier.
* Login to your Docker registry.
* In the project root directory create `dappfile.yaml` with the same content as on the corresponding step earlier.

Check, that if you try to build the image, dapp won't use stages cache and will build all stages. Execute:

```
dapp dimg build --dry-run
```

Dapp won't do any changes but you will get an output with `[BUILD]` against every stage. It shows that there are no images in your local cache and every stage will be rebuild.

### Pulling stages cache

To use stages cache which you pushed from `node1`, you need to pull them before building.

Execute the following command (on `node2`):

```
dapp dimg stages pull <registry_address>
```

> Dapp looks into dappfile, calculates signatures of stages and pulls only last images existing in Docker registry, according to stages conveyor. If you need to pull images of every stage you can use `--all` option with `dapp dimg stages pull` command (don't do it on this step or you will get the result which will be different from this example)

### Building the application

To build the image, execute the following command (on `node2`):
```
dapp dimg build
```

You will get an output with `[NOT PRESENT]` against some stages and `[USING CACHE]` against the last stage like this (stages signature was shortened):
```
From                                                   [NOT PRESENT]
  signature: dimgstage-symfony-demo:41772...9a11
Before install                                         [NOT PRESENT]
  signature: dimgstage-symfony-demo:95961...cbde
Git artifacts: create archive                          [NOT PRESENT]
  signature: dimgstage-symfony-demo:49b8b...85c4
Install group
  Git artifacts: apply patches (before install)        [NOT PRESENT]
    signature: dimgstage-symfony-demo:52c41...3add
  Install                                              [NOT PRESENT]
    signature: dimgstage-symfony-demo:39db3...24fb
Setup group
  Git artifacts: apply patches (before setup)          [NOT PRESENT]
    signature: dimgstage-symfony-demo:c1220...ed2b
  Setup                                                [NOT PRESENT]
    signature: dimgstage-symfony-demo:d3d21...0e70
  Git artifacts: apply patches (after setup)           [NOT PRESENT]
    signature: dimgstage-symfony-demo:18144...e548
    date: 2018-10-30 15:32:18 +0300
    size: 457.613 MB
Docker instructions                                    [USING CACHE]
  signature: dimgstage-symfony-demo:bda2d7a...160e
  date: 2018-10-30 15:32:21 +0300
  difference: 0.0 MB
Running time 9.74 seconds
```

Pay attention to the `Running time` — using cache takes time to pull images from Docker registry but reduce time to build.

> If you used `--all` option with the `dapp dimg stages pull` command, you get `[USING CACHE]` against every stage instead of `[NOT PRESENT]` in output.

### Making changes and rebuilding the application on the node2

Open the `dappfile.yaml` and add the following string to the `Setup` stage:
```
- raw: echo "Some changes"
```

Your `Setup` stage should look like this:

```shell
setup:
- name: "Create start script"
  copy:
    content: |
      #!/bin/bash
      php bin/console server:run 0.0.0.0:8000
    dest: /app/start.sh
    owner: app
    group: app
    mode: 0755
- raw: echo `date` > /app/version.txt
- raw: chown app:app /app/version.txt
- raw: echo "Some changes"
```

If you try to build the image now, dapp will build every stage, because you don't have necessary subordinate layers in your local cache (if you didn't use `--all` option with the `dapp dimg stages pull` command). To pull necessary layers you  need to pull stages cache after changes you made, before a build.

Pull cache according to your modified dappfile, build image and push it with the stages cache:

```
dapp dimg stages pull <registry_address>
dapp dimg bp --with-stages <registry_address>
```

Dapp pulls stage prior to the `Setup` stage, then build the image with the stages starting from the `Setup` stage, then push image and stages cache to the Docker registry.

## Conclusions

Dapp can use a distributed cache and can work with:
* **Dynamic resources.** You can start build node on-demand, pull stages cache and build n image.
* **More than one build nodes.** You can have more than one build nodes in your environment.

The only steps you need are:
* pull existing stages cache from the Docker registry with the `dapp dimg stages pull REPO` command (before building!);
* build images and push it with stages cache to the Docker registry with the `dapp dimg bp --with-stages REPO` command.
