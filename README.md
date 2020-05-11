## On CKAN BASE

This branch ONLY uses ckan-dev:2.8 image from OKF.

## Note on requirements

We use a `requirements.txt` file to easily install add new requirements for development.
However, this installation uses the `requirements-freeze.txt` to instantiate the local development environment and the test environment (TODO check if this true).

Use `make requirements` to generate a `requirements-freeze.txt` file that is up to date with `requirements.txt`

# Docker Compose setup for CKAN

* [Overview](#overview)
* [Quick start](#quick-start)
* [Development mode](#development-mode)
   * [Create an extension](#create-an-extension)
   * [Running the debugger (pdb / ipdb)](#running-the-debugger-pdb--ipdb)
* [CKAN images](#ckan-images)
   * [Extending the base images](#extending-the-base-images)
   * [Applying patches](#applying-patches)
* [Known Issues](#known-issues)


## Overview

Local development environment for catalog.data.gov's catalog-next

Based on [Open Knowledge Foundation](https://okfn.org/)'s [docker-ckan](https://github.com/okfn/docker-ckan) project.

It includes the following images, all based on [Alpine Linux](https://alpinelinux.org/):

* CKAN: modified from keitaro/ckan (see [CKAN Images](#ckan-images)) for more details). File uploads are stored in a named volume.
* DataPusher: modified from keitaro/datapusher
* PostgreSQL: Official PostgreSQL image. Database files are stored in a named volume.
* Solr: official Solr image with CKAN's schema. Index data is stored in a named volume.
* Redis: standard Redis image

The site is configured via env vars (the base CKAN image loads [ckanext-envvars](https://github.com/okfn/ckanext-envvars)), that you can set in the `.env` file.

## Quick start

Copy the included `.env.example` and rename it to `.env` to modify it depending on your own needs.

Using the default values on the `.env.example` file will get you a working Catalog Next instance. There is a sysadmin user created by default with the values defined in `CKAN_SYSADMIN_NAME` and `CKAN_SYSADMIN_PASSWORD`(`ckan_admin` and `test1234` by default). I shouldn't be telling you this but obviously don't run any public CKAN instance with the default settings.

To build the images:

        `make build`

To start the containers:
        `make up`

To remove the containers and volumes:

        `make clean`

## Development mode

To build the images:

        `make build`

To start the containers in development mode:

        `make dev`

To remove the containers and volumes:

        `make build dev`


See [CKAN Images](#ckan-images) for more details of what happens when using development mode.


### Create an extension

You can use the paster template in much the same way as a source install, only executing the command inside the CKAN container and setting the mounted `src/` folder as output:

    docker-compose -f docker-compose.dev.yml exec ckan-dev /bin/bash -c "paster --plugin=ckan create -t ckanext ckanext-myext -o /srv/app/src_extensions"

The new extension will be created in the `src/` folder. You might need to change the owner of its folder to have the appropiate permissions.


### Running the debugger (pdb / ipdb)

To run a container and be able to add a breakpoint with `pdb` or `ipdb`, run the `ckan-dev` container with the `--service-ports` option:

    docker-compose -f docker-compose.dev.yml run --service-ports ckan-dev

This will start a new container, displaying the standard output in your terminal. If you add a breakpoint in a source file in the `src` folder (`import pdb; pdb.set_trace()`) you will be able to inspect it in this terminal next time the code is executed.


## CKAN images

```
    +-------------------------+                +----------+
    |                         |                |          |
    | openknowledge/ckan-base +---------------->   ckan   | (production)
    |                         |                |          |
    +-----------+-------------+                +----------+
                |
                |
    +-----------v------------+                 +----------+
    |                        |                 |          |
    | openknowledge/ckan-dev +----------------->   ckan   | (development)
    |                        |                 |          |
    +------------------------+                 +----------+


```

The Docker images used to build your CKAN project are located in the `ckan/` folder. There are two Docker files:

* `Dockerfile`: this is based on `openknowledge/ckan-base` (with the `Dockerfile` on the `/ckan-base/<version>` folder), an image with CKAN with all its dependencies, properly configured and running on [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/) (production setup)
* `Dockerfile.dev`: this is based on `openknowledge/ckan-dev` (with the `Dockerfile` on the `/ckan-dev/<version>` folder), wich extends `openknowledge/ckan-base` to include:

  * Any extension cloned on the `src` folder will be installed in the CKAN container when booting up Docker Compose (`docker-compose up`). This includes installing any requirements listed in a `requirements.txt` (or `pip-requirements.txt`) file and running `python setup.py develop`.
  * The CKAN image used will development requirements needed to run the tests .
  * CKAN will be started running on the paster development server, with the `--reload` option to watch changes in the extension files.
  * Make sure to add the local plugins to the `CKAN__PLUGINS` env var in the `.env` file.

From these two base images you can build your own customized image tailored to your project, installing any extensions and extra requirements needed.

### Extending the base images

To perform extra initialization steps you can add scripts to your custom images and copy them to the `/docker-entrypoint.d` folder (The folder should be created for you when you build the image). Any `*.sh` and `*.py` file in that folder will be executed just after the main initialization script ([`prerun.py`](https://github.com/okfn/docker-ckan/blob/master/ckan-base/setup/prerun.py)) is executed and just before the web server and supervisor processes are started.

For instance, consider the following custom image:

```
ckan
├── docker-entrypoint.d
│   └── setup_validation.sh
├── Dockerfile
└── Dockerfile.dev

```

We want to install an extension like [ckanext-validation](https://github.com/frictionlessdata/ckanext-validation) that needs to create database tables on startup time. We create a `setup_validation.sh` script in a `docker-entrypoint.d` folder with the necessary commands:

```bash
#!/bin/bash

# Create DB tables if not there
paster --plugin=ckanext-validation validation init-db -c $CKAN_INI
```

And then in our `Dockerfile` we install the extension and copy the initialization scripts:

```Dockerfile
FROM openknowledge/ckan-dev:2.7

RUN pip install -e git+https://github.com/frictionlessdata/ckanext-validation.git#egg=ckanext-validation && \
    pip install -r https://raw.githubusercontent.com/frictionlessdata/ckanext-validation/master/requirements.txt

COPY docker-entrypoint.d/* /docker-entrypoint.d/
```

### Applying patches

When building your project specific CKAN images (the ones defined in the `ckan/` folder), you can apply patches 
to CKAN core or any of the built extensions. To do so create a folder inside `ckan/patches` with the name of the
package to patch (ie `ckan` or `ckanext-??`). Inside you can place patch files that will be applied when building
the images. The patches will be applied in alphabetical order, so you can prefix them sequentially if necessary.

For instance, check the following example image folder:

```
ckan
├── patches
│   ├── ckan
│   │   ├── 01_datasets_per_page.patch
│   │   ├── 02_groups_per_page.patch
│   │   ├── 03_or_filters.patch
│   └── ckanext-harvest
│       └── 01_resubmit_objects.patch
├── Dockerfile
└── Dockerfile.dev

```

## Tests

Run integration / smoke tests using:

      `make test`

## CI

Continuous Integration via [Circle CI](https://app.circleci.com/pipelines/github/GSA/catalog.data.gov)

