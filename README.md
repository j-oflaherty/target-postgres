# Target Postgres

[![CircleCI](https://circleci.com/gh/datamill-co/target-postgres.svg?style=svg)](https://circleci.com/gh/datamill-co/target-postgres)

A [Singer](https://singer.io/) postgres target, for use with Singer streams generated by Singer taps.

## Features

- Creates SQL tables for [Singer](https://singer.io) streams
- Denests objects flattening them into the parent object's table
- Denests rows into separate tables
- Adds columns and sub-tables as new fields are added to the stream [JSON Schema](https://json-schema.org/)
- Full stream replication via record `version` and `ACTIVATE_VERSION` messages.

## Install

```sh
pip install singer-target-postgres
```

## Usage

1. Follow the
   [Singer.io Best Practices](https://github.com/singer-io/getting-started/blob/master/docs/RUNNING_AND_DEVELOPING.md#running-a-singer-tap-with-a-singer-target)
   for setting up separate `tap` and `target` virtualenvs to avoid version
   conflicts.

1. Create a [config file](#configjson) at
   `~/singer.io/target_postgres_config.json` with postgres connection
   information and target postgres schema.

   ```json
   {
     "postgres_host": "localhost",
     "postgres_port": 5432,
     "postgres_database": "my_analytics",
     "postgres_username": "myuser",
     "postgres_password": "1234",
     "postgres_schema": "mytapname"
   }
   ```

1. Run `target-postgres` against a [Singer](https://singer.io) tap.

   ```bash
   ~/.virtualenvs/tap-something/bin/tap-something \
     | ~/.virtualenvs/target-postgres/bin/target-postgres \
       --config ~/singer.io/target_postgres_config.json
   ```

### Config.json

The fields available to be specified in the config file are specified
here.

| Field | Type | Default | Details |
| ----- | ---- | ------- | ------- |
| `postgres_host` |`["string", "null"]` | `"localhost"` | |
| `postgres_port` | `["integer", "null"]`|  `5432` | |
| `postgres_database` | `["string"]`|  `N/A` | |
| `postgres_username` | `["string", "null"]` |  `N/A` | |
| `postgres_password` | `["string", "null"]`|  `null` | |
| `postgres_schema` | `["string", "null"]` |  `"public"` | |
| `invalid_records_detect` | `["boolean", "null"]`| `true` | Include `false` in your config to disable `target-postgres` from crashing on invalid records |
| `invalid_records_threshold` | `["integer", "null"]` | `0` | Include a positive value `n` in your config to allow for `target-postgres` to encounter at most `n` invalid records per stream before giving up. |
| `disable_collection` | `["string", "null"]` | `false` | Include `true` in your config to disable [Singer Usage Logging](#usage-logging).

### Supported Versions

`target-postgres` only supports [JSON Schema Draft4](http://json-schema.org/specification-links.html#draft-4).
While declaring a schema _is optional_, any input schema which declares a version
other than 4 will be rejected.

`target-postgres` supports all versions of PostgreSQL which are presently supported
by the PostgreSQL Global Development Group. Our [CI config](https://github.com/datamill-co/target-postgres/blob/master/.circleci/config.yml) defines all versions we are currently supporting.


| Version | Current minor | Supported | First Release | Final Release |
| ------- | ------------- | --------- | ------------- | ------------- |
| 11 | 11.1 | Yes | October 18, 2018 | November 9, 2023 |
| 10 | 10.6 | Yes | October 5, 2017 | November 10, 2022 |
| 9.6 | 9.6.11 | Yes | September 29, 2016 | November 11, 2021 |
| 9.5 | 9.5.15 | Yes | January 7, 2016 | February 11, 2021 |
| 9.4 | 9.4.20 | Yes | December 18, 2014 | February 13, 2020 |
| 9.3 | 9.3.25 | No | September 9, 2013 | November 8, 2018 |

_The above is copied from the [current list of versions](https://www.postgresql.org/support/versioning/) on Postgresql.org_

## Known Limitations

- Ignores `STATE` Singer messages.
- Requires a [JSON Schema](https://json-schema.org/) for every stream.
- Only string, string with date-time format, integer, number, boolean,
  object, and array types with or without null are supported. Arrays can
  have any of the other types listed, including objects as types within
  items.
    - Example of JSON Schema types that work
        - `['number']`
        - `['string']`
        - `['string', 'null']`
    - Exmaple of JSON Schema types that **DO NOT** work
        - `['string', 'integer']`
        - `['integer', 'number']`
        - `['any']`
        - `['null']`
- JSON Schema combinations such as `anyOf` and `allOf` are not supported.
- JSON Schema $ref is partially supported:
  - ***NOTE:*** The following limitations are known to **NOT** fail gracefully
  - Presently you cannot have any circular or recursive `$ref`s
  - `$ref`s must be present within the schema:
    - URI's do not work
    - if the `$ref` is broken, the behaviour is considered unexpected
- Any values which are the `string` `NULL` will be streamed to PostgreSQL as the literal `null`
- Table names are restricted to:
  - 63 characters in length
  - can only be composed of `_`, lowercase letters, numbers, `$`
  - cannot start with `$`
  - ASCII characters
- Field/Column names are restricted to:
  - 63 characters in length
  - ASCII characters

## Usage Logging

[Singer.io](https://www.singer.io/) requires official taps and targets to collect anonymous usage data. This data is only used in aggregate to report on individual tap/targets, as well as the Singer community at-large. IP addresses are recorded to detect unique tap/targets users but not shared with third-parties.

To disable anonymous data collection set `disable_collection` to `true` in the configuration JSON file.

## Developing

`target-postgres` utilizes [setup.py](https://python-packaging.readthedocs.io/en/latest/index.html) for package
management, and [PyTest](https://docs.pytest.org/en/latest/contents.html) for testing.

### Documentation

See also:

- [DECISIONS](./DECISIONS.md): A document containing high level explanations of various decisions and decision making paradigms. A good place to request more explanation/clarification on confusing things found herein.
- [TableMetadata](./docs/TableMetadata.md): A document detailing some of the metadata necessary for `TargetPostgres` to function correctly on the Remote

### Docker

If you have [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) installed, you can
easily run the following to get a local env setup quickly.

```sh
$ docker-compose up -d --build
$ docker logs -tf target-postgres_target-postgres_1 # You container names might differ
```

As soon as you see `INFO: Dev environment ready.` you can shell into the container and start running test commands:

```sh
$ docker exec -it target-postgres_target-postgres_1 bash # Your container names might differ
```

See the [PyTest](#pytest) commands below!

### DB

To run the tests, you will need a PostgreSQL server running.

***NOTE:*** Testing assumes that you've exposed the traditional port `5432`.

Make sure to set the following env vars for [PyTest](#pytest):

```sh
$ EXPORT POSTGRES_HOST='<your-host-name>' # Most likely 'localhost'
$ EXPORT POSTGRES_DB='<your-db-name>'     # We use 'target_postgres_test'
$ EXPORT POSTGRES_USER='<your-user-name'  # Probably just 'postgres', make sure this user has no auth
```

### PyTest

To run tests, try:

```sh
$ python setup.py pytest
```

If you've `bash` shelled into the Docker Compose container ([see above](#docker)), you should be able to simply use:

```sh
$ pytest
```

## Sponsorship

Target Postgres is sponsored by Data Mill (Data Mill Services, LLC) [datamill.co](https://datamill.co/).

Data Mill helps organizations utilize modern data infrastructure and data science to power analytics, products, and services.

------
Copyright Data Mill Services, LLC 2018
