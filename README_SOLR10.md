# Solr 10 Notes

This project is legacy code and does not run on Solr 10 unchanged.

The current repo state was verified against a disposable Solr `10.0.0`
instance under `/tmp` with the compatibility fixes in this branch.

## What worked

The plugin ran successfully in a local `/tmp` Solr 10 instance when started
with:

- `-Dsolr.config.lib.enabled=true`
- `-Dsolr.sharedLib=...`
- `SOLR_SECURITY_MANAGER_ENABLED=false`

The last setting matters. Without disabling the Solr security manager, the
legacy Velocity sandbox hit permission failures similar to the behavior
described in:

- <https://github.com/erikhatcher/solr-velocity/issues/5>

## Disposable test layout

The Solr install and runtime files were kept out of the repo:

- Solr install: `/tmp/solr-10.0.0`
- Solr home: `/tmp/solr-velocity-run/home`
- Shared lib dir: `/tmp/solr-velocity-run/lib`
- Logs: `/tmp/solr-velocity-run/logs`

The shared lib dir contained:

- `solritas-0.9.5.jar`
- `velocity-engine-core-2.1.jar`
- `velocity-tools-generic-3.0.jar`

## Build

Build the jar from the repo with:

```bash
export LANG=C.UTF-8 LC_ALL=C.UTF-8
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH="$JAVA_HOME/bin:$PATH"
mvn -Dmaven.test.skip=true package
```

Result:

- `target/solritas-0.9.5.jar`

## Prepare the Solr home

Create a disposable Solr home and copy the test core config from this repo:

```bash
mkdir -p /tmp/solr-velocity-run/home/docsearch_core/conf
mkdir -p /tmp/solr-velocity-run/lib
mkdir -p /tmp/solr-velocity-run/logs

cp -r src/test-files/solr/collection1/conf/* \
  /tmp/solr-velocity-run/home/docsearch_core/conf/
cp /tmp/solr-10.0.0/server/solr/solr.xml \
  /tmp/solr-velocity-run/home/solr.xml

printf 'name=docsearch_core\n' \
  > /tmp/solr-velocity-run/home/docsearch_core/core.properties
```

Copy the plugin jar and its runtime dependencies:

```bash
cp target/solritas-0.9.5.jar /tmp/solr-velocity-run/lib/
cp ~/.m2/repository/org/apache/velocity/velocity-engine-core/2.1/velocity-engine-core-2.1.jar \
  /tmp/solr-velocity-run/lib/
cp ~/.m2/repository/org/apache/velocity/tools/velocity-tools-generic/3.0/velocity-tools-generic-3.0.jar \
  /tmp/solr-velocity-run/lib/
```

Notes:

- The old `<lib/>` entry inside `solrconfig.xml` is ignored by Solr 10.
- `solr.sharedLib` is what actually loaded the jars in this test.

## Start Solr 10

Start Solr in standalone mode:

```bash
SOLR_SECURITY_MANAGER_ENABLED=false \
/tmp/solr-10.0.0/bin/solr start -f --user-managed -p 8990 \
  --server-dir /tmp/solr-10.0.0/server \
  --solr-home /tmp/solr-velocity-run/home \
  --jvm-opts '-Dsolr.logs.dir=/tmp/solr-velocity-run/logs -Dsolr.config.lib.enabled=true -Dsolr.sharedLib=/tmp/solr-velocity-run/lib'
```

## Verify

This request returned HTTP 200 in the `/tmp` Solr 10 test:

```bash
curl -i -sS \
  'http://127.0.0.1:8990/api/cores/docsearch_core/search?q=*:*&wt=velocity&v.template=numFound'
```

Expected response shape:

```http
HTTP/1.1 200 OK
Content-Type: text/html;charset=utf-8

0
```

## Notes on remaining rough edges

- This was verified with test-skipping Maven packaging:
  `mvn -Dmaven.test.skip=true package`
- Solr 10 logs still show Velocity property deprecation warnings.
- Running with the Solr security manager enabled likely still requires extra
  permission work in the plugin sandbox.
