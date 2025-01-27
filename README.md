# Martin

[![CircleCI](https://img.shields.io/circleci/project/github/urbica/martin.svg?style=popout)](https://circleci.com/gh/urbica/martin)
[![Docker pulls](https://img.shields.io/docker/pulls/urbica/martin.svg)](https://hub.docker.com/r/urbica/martin)
[![Metadata](https://images.microbadger.com/badges/image/urbica/martin.svg)](https://microbadger.com/images/urbica/martin)

Martin is a [PostGIS](https://github.com/postgis/postgis) [vector tiles](https://github.com/mapbox/vector-tile-spec) server suitable for large databases. Martin is written in [Rust](https://github.com/rust-lang/rust) using [Actix](https://github.com/actix/actix-web) web framework.

![Martin](https://raw.githubusercontent.com/urbica/martin/master/logo.png)

- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [API](#api)
- [Using with Mapbox GL JS](#using-with-mapbox-gl-js)
- [Table Sources](#table-sources)
  - [Table Sources List](#table-sources-list)
  - [Table Source TileJSON](#table-source-tilejson)
  - [Table Source Tiles](#table-source-tiles)
- [Function Sources](#function-sources)
  - [Function Sources List](#function-sources-list)
  - [Function Source TileJSON](#function-source-tilejson)
  - [Function Source Tiles](#function-source-tiles)
- [Command-line Interface](#command-line-interface)
- [Environment Variables](#environment-variables)
- [Configuration File](#configuration-file)
- [Using with Docker](#using-with-docker)
- [Using with Docker Compose](#using-with-docker-compose)
- [Using with Nginx](#using-with-nginx)
- [Building from Source](#building-from-source)
- [Debugging](#debugging)
- [Development](#development)

## Requirements

Martin requires PostGIS >= 2.4.0.

## Installation

You can download martin from [Github releases page](https://github.com/urbica/martin/releases).

If you are using macOS and [Homebrew](https://brew.sh/) you can install martin using Homebrew tap.

```shell
brew tap urbica/tap
brew install martin
```

## Usage

Martin requires a database connection string. It can be passed as a command-line argument or as a `DATABASE_URL` environment variable.

```shell
martin postgres://postgres@localhost/db
```

## API

| Method | URL                                                  | Description                                           |
| ------ | ---------------------------------------------------- | ----------------------------------------------------- |
| `GET`  | `/index.json`                                        | [Table Sources List](#table-sources-list)             |
| `GET`  | `/{schema_name}.{table_name}.json`                   | [Table Source TileJSON](#table-source-tilejson)       |
| `GET`  | `/{schema_name}.{table_name}/{z}/{x}/{y}.pbf`        | [Table Source Tiles](#table-source-tiles)             |
| `GET`  | `/rpc/index.json`                                    | [Function Sources List](#function-sources-list)       |
| `GET`  | `/rpc/{schema_name}.{function_name}.json`            | [Function Source TileJSON](#function-source-tilejson) |
| `GET`  | `/rpc/{schema_name}.{function_name}/{z}/{x}/{y}.pbf` | [Function Source Tiles](#function-source-tiles)       |

## Using with Mapbox GL JS

[Mapbox GL JS](https://github.com/mapbox/mapbox-gl-js) is a JavaScript library for interactive, customizable vector maps on the web. It takes map styles that conform to the
[Mapbox Style Specification](https://www.mapbox.com/mapbox-gl-js/style-spec), applies them to vector tiles that
conform to the [Mapbox Vector Tile Specification](https://github.com/mapbox/vector-tile-spec), and renders them using
WebGL.

You can add a layer to the map and specify martin TileJSON endpoint as a vector source URL. You should also specify a `source-layer` property. For [Table Sources](#table-sources) it is `{schema_name}.{table_name}` by default.

```js
map.addLayer({
  id: 'public.points',
  type: 'circle',
  source: {
    type: 'vector',
    url: 'http://localhost:3000/public.points.json'
  },
  'source-layer': 'public.points'
});
```

## Table Sources

Table Source is a database table which can be used to query [vector tiles](https://github.com/mapbox/vector-tile-spec). When started, martin will go through all spatial tables in the database and build a list of table sources. A table should have at least one geometry column with non-zero SRID. All other table columns will be represented as properties of a vector tile feature.

### Table Sources List

Table Sources list endpoint is available at `/index.json`

```shell
curl localhost:3000/index.json
```

**Note**: if in `watch` mode, this will rescan database for table sources.

### Table Source TileJSON

Table Source [TileJSON](https://github.com/mapbox/tilejson-spec) endpoint is available at `/{schema_name}.{table_name}.json`.

For example, `points` table in `public` schema will be available at `/public.points.json`

```shell
curl localhost:3000/public.points.json
```

### Table Source Tiles

Table Source tiles endpoint is available at `/{schema_name}.{table_name}/{z}/{x}/{y}.pbf`

For example, `points` table in `public` schema will be available at `/public.points/{z}/{x}/{y}.pbf`

```shell
curl localhost:3000/public.points/0/0/0.pbf
```

## Function Sources

Function Source is a database function which can be used to query [vector tiles](https://github.com/mapbox/vector-tile-spec). When started, martin will look for the functions with a suitable signature. A function that takes `z integer`, `x integer`, `y integer`, and `query_params json` and returns `bytea`, can be used as a Function Source.

| Argument     | Type    | Description             |
| ------------ | ------- | ----------------------- |
| z            | integer | Tile zoom parameter     |
| x            | integer | Tile x parameter        |
| y            | integer | Tile y parameter        |
| query_params | json    | Query string parameters |

**Hint**: You may want to use [TileBBox](https://github.com/mapbox/postgis-vt-util#tilebbox) function to generate bounding-box geometry of the area covered by a tile.

Here is an example of a function that can be used as a Function Source.

```sql
CREATE OR REPLACE FUNCTION public.function_source(z integer, x integer, y integer, query_params json) RETURNS BYTEA AS $$
DECLARE
  bounds GEOMETRY(POLYGON, 3857) := TileBBox(z, x, y, 3857);
  mvt BYTEA;
BEGIN
  SELECT INTO mvt ST_AsMVT(tile, 'public.function_source', 4096, 'geom') FROM (
    SELECT
      ST_AsMVTGeom(geom, bounds, 4096, 64, true) AS geom
    FROM public.table_source
    WHERE geom && bounds
  ) as tile WHERE geom IS NOT NULL;

  RETURN mvt;
END
$$ LANGUAGE plpgsql IMMUTABLE STRICT PARALLEL SAFE;
```

### Function Sources List

Function Sources list endpoint is available at `/rpc/index.json`

```shell
curl localhost:3000/rpc/index.json
```

**Note**: if in `watch` mode, this will rescan database for function sources.

### Function Source TileJSON

Function Source [TileJSON](https://github.com/mapbox/tilejson-spec) endpoint is available at `/rpc/{schema_name}.{function_name}.json`

For example, `points` function in `public` schema will be available at `/rpc/public.points.json`

```shell
curl localhost:3000/rpc/public.points.json
```

### Function Source Tiles

Function Source tiles endpoint is available at `/rpc/{schema_name}.{function_name}/{z}/{x}/{y}.pbf`

For example, `points` function in `public` schema will be available at `/rpc/public.points/{z}/{x}/{y}.pbf`

```shell
curl localhost:3000/rpc/public.points/0/0/0.pbf
```

## Command-line Interface

You can configure martin using command-line interface

```shell
Usage:
  martin [options] [<connection>]
  martin -h | --help
  martin -v | --version

Options:
  -h --help               Show this screen.
  -v --version            Show version.
  --config=<path>         Path to config file.
  --keep_alive=<n>        Connection keep alive timeout [default: 75].
  --listen_addresses=<n>  The socket address to bind [default: 0.0.0.0:3000].
  --pool_size=<n>         Maximum connections pool size [default: 20].
  --watch                 Scan for new sources on sources list requests
  --workers=<n>           Number of web server workers.
```

## Environment Variables

You can also configure martin using environment variables

| Environment variable | Example                          | Description                   |
| -------------------- | -------------------------------- | ----------------------------- |
| DATABASE_URL         | postgres://postgres@localhost/db | postgres database connection  |
| DATABASE_POOL_SIZE   | 20                               | maximum connections pool size |
| KEEP_ALIVE           | 75                               | connection keep alive timeout |
| WATCH_MODE           | true                             | scan for new sources          |
| WORKER_PROCESSES     | 8                                | number of web server workers  |

## Configuration File

If you don't want to expose all of your tables and functions, you can list your sources in a configuration file. To start martin with a configuration file you need to pass a path to a file with a `--config` argument.

```shell
martin --config config.yaml
```

You can find an example of a configuration file [here](https://github.com/urbica/martin/blob/master/tests/config.yaml).

```yaml
# Database connection string
connection_string: 'postgres://postgres@localhost/db'

# Maximum connections pool size [default: 20]
pool_size: 20

# Connection keep alive timeout [default: 75]
keep_alive: 75

# Number of web server workers
worker_processes: 8

# The socket address to bind [default: 0.0.0.0:3000]
listen_addresses: '0.0.0.0:3000'

# Enable watch mode
watch: true

# associative arrays of table sources
table_sources:
  public.table_source:
    # table source id
    id: public.table_source

    # table schema
    schema: public

    # table name
    table: table_source

    # geometry column name
    geometry_column: geom

    # geometry srid
    srid: 4326

    # tile extent in tile coordinate space
    extent: 4096

    # buffer distance in tile coordinate space to optionally clip geometries
    buffer: 64

    # boolean to control if geometries should be clipped or encoded as is
    clip_geom: true

    # geometry type
    geometry_type: GEOMETRY

    # list of columns, that should be encoded as a tile properties
    properties:
      gid: int4

# associative arrays of function sources
function_sources:
  public.function_source:
    # function source id
    id: public.function_source

    # schema name
    schema: public

    # function name
    function: function_source
```

## Using with Docker

You can use official Docker image [`urbica/martin`](https://hub.docker.com/r/urbica/martin)

```shell
docker run \
  -p 3000:3000 \
  -e DATABASE_URL=postgres://postgres@localhost/db \
  urbica/martin
```

If you are running PostgreSQL instance on `localhost`, you have to change network settings to allow the Docker container to access the `localhost` network.

For Linux, add the `--net=host` flag to access the `localhost` PostgreSQL service.

```shell
docker run \
  --net=host \
  -p 3000:3000 \
  -e DATABASE_URL=postgres://postgres@localhost/db \
  urbica/martin
```

For macOS, use `host.docker.internal` as hostname to access the `localhost` PostgreSQL service.

```shell
docker run \
  -p 3000:3000 \
  -e DATABASE_URL=postgres://postgres@host.docker.internal/db \
  urbica/martin
```

For Windows, use `docker.for.win.localhost` as hostname to access the `localhost` PostgreSQL service.

```shell
docker run \
  -p 3000:3000 \
  -e DATABASE_URL=postgres://postgres@docker.for.win.localhost/db \
  urbica/martin
```

## Using with Docker Compose

You can use example [`docker-compose.yml`](https://raw.githubusercontent.com/urbica/martin/master/docker-compose.yml) file as a reference

```yml
version: '3'

services:
  martin:
    image: urbica/martin
    restart: unless-stopped
    ports:
      - 3000:3000
    environment:
      - WATCH_MODE=true
      - DATABASE_URL=postgres://postgres@db/db
    depends_on:
      - db

  db:
    image: mdillon/postgis:11-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=postgres
    volumes:
      - ./pg_data:/var/lib/postgresql/data
```

First, you need to start `db` service

```shell
docker-compose up -d db
```

Then, after `db` service is ready to accept connections, you can start `martin`

```shell
docker-compose up -d martin
```

By default, martin will be available at [localhost:3000](http://localhost:3000/index.json)

## Using with Nginx

If you are running martin behind nginx proxy, you may want to rewrite request URL, to properly handle tile urls in [TileJSON](#table-source-tilejson) [endpoints](#function-source-tilejson).

```nginx
location ~ /tiles/(?<fwd_path>.*) {
    proxy_set_header  X-Rewrite-URL $request_uri;
    proxy_set_header  X-Forwarded-Host $host;
    proxy_pass        http://martin:3000/$fwd_path$is_args$args;
}
```

## Building from Source

You can clone the repository and build martin using [cargo](https://doc.rust-lang.org/cargo) package manager.

```shell
git clone git@github.com:urbica/martin.git
cd martin
cargo build --release
```

The binary will be available at `./target/release/martin`.

```shell
cd ./target/release/
./martin postgres://postgres@localhost/db
```

## Debugging

Log levels are controlled on a per-module basis, and by default all logging is disabled except for errors. Logging is controlled via the `RUST_LOG` environment variable. The value of this environment variable is a comma-separated list of logging directives.

This will enable verbose logging for the `actix_web` module and enable debug logging for the `martin` and `postgres` modules:

```shell
export RUST_LOG=actix_web=info,martin=debug,postgres=debug
martin postgres://postgres@localhost/db
```

## Development

Install project dependencies and check if all the tests are running.

```shell
cargo test
cargo run
```
