> [!Note]
> As of [immich 1.133.0](https://github.com/immich-app/immich/releases/tag/v1.133.0), pgvecto.rs is being deprecated in favor of [VectorChord](https://docs.vectorchord.ai/).  Therefore, this repo is no longer required and is archived.  For bitnami images with VectorChord, please visit https://github.com/aaronspruit/bitnami-pg-vectorchord.  


# Bitnami Postgres images w/ pgvecto.rs

I'm using the [Bitnami postgres helm](https://github.com/bitnami/charts/blob/main/bitnami/postgresql/README.md) chart as my backing store for immich (not using the immich helm chart).  Immich requires [pgvecto.rs](https://pgvecto.rs/) installed, which isn't part of the bitnami image.  This image and the subseqent configs (listed below) are how I use this.

## Usage

This is a drop in replacement for bitnami postgres debian container for the bitnami postgres helm chart. Just swap the image with mine and if you're using this for immich, you'll need to following configs.

In your `values.yaml`:

```yaml
image:
  registry: ghcr.io
  repository: aaronspruit/bitnami-pg-pgvecto-rs
  tag: pg16.4.0-v0.2.1-v2
primary:
  # this adds the libraries
  extendedConfiguration: |-
    shared_preload_libraries = 'vectors.so'
  initdb:
    user: postgres
    scripts: 
      # this script installs the extension
      # and then updates the database that was already created
      # https://github.com/immich-app/immich/discussions/7252#discussioncomment-8534336
      # https://docs.pgvecto.rs/getting-started/installation.html#from-debian-package 
      00-create-extensions.sql: |
        \getenv dbname POSTGRES_DATABASE
        \getenv dbuser POSTGRES_USER
        \c :dbname
        ALTER SYSTEM SET search_path TO "$user", public, vectors;
        CREATE EXTENSION IF NOT EXISTS cube;
        CREATE EXTENSION IF NOT EXISTS earthdistance;
        CREATE EXTENSION IF NOT EXISTS vectors;
        ALTER DATABASE :dbname OWNER TO :dbuser;
        GRANT ALL ON SCHEMA vectors TO :dbuser;
      # need to restart to finish loading pgvecto.rs
      01-restart.sh: |
        #!/bin/sh
        pg_ctl restart
      # after restart, the index_stat table is created, so need to modify that now too
      02-grant-index.sql: |
        \getenv dbname POSTGRES_DATABASE
        \getenv dbuser POSTGRES_USER
        \c :dbname
        GRANT SELECT ON TABLE pg_vector_index_stat to :dbuser;
```
