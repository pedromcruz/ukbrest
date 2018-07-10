[![Build Status](https://travis-ci.org/hakyimlab/ukbrest.svg?branch=master)](https://travis-ci.org/hakyimlab/ukbrest)
[![Coverage Status](https://coveralls.io/repos/github/miltondp/ukbrest/badge.svg?branch=master)](https://coveralls.io/github/miltondp/ukbrest?branch=master)

# ukbrest

`TODO`: this documentation is still work in progress. It will be updated and completed soon.

## Downloading and running
The easiest way to get ukbrest up and running is using Docker. The instructions below are just to get things working
quickly. You should read the Docker documentation if you want to, for example, keep data saved across different runs.

**The first thing** you have to do is creating a virtual network and getting PostgreSQL up:

```bash
$ docker network create ukb

$ docker run -d --name pg --net ukb -e POSTGRES_USER=test \
  -e POSTGRES_PASSWORD=test -e POSTGRES_DB=ukb -p 5432:5432 \
  postgres:9.6
```

**Secondly**, you have to load your phenotype data into PostgreSQL:

```bash
$ docker run --rm --net ukb \
  -v /mnt/ukbrest/genotype:/var/lib/genotype \
  -v /mnt/ukbrest/phenotype/:/var/lib/phenotype \
  -e UKBREST_GENOTYPE_BGEN_SAMPLE_FILE="ukb1952_v2_s487398.sample" \
  -e UKBREST_DB_URI="postgresql://test:test@pg:5432/ukb" \
  miltondp/ukbrest --load
```

The BGEN sample file (specified by the environment variable `UKBREST_GENOTYPE_BGEN_SAMPLE_FILE`) is
relative to the genotype directory (`/mnt/ukbrest/genotype`, in the example above).

Once the loading process finishes, you can get all the data field codings by connecting to the
PostgreSQL database and exporting a list of codings:

```sql
\copy (select distinct coding from fields where coding is not null)
to /mnt/all_codings.txt (format csv)
```

The file `/mnt/all_codings.txt` is just a list of coding numbers, one per line, that you can use
to download all coding files using the `download_codings.sh` script:

```bash
$ mkdir /tmp/codings && cd /mnt/codings
$ [...]/misc/download_codings.sh all_codings.txt
```

`TODO`: how to load coding tables.

**The third step** consists in indexing your genotype data. You have to use the [bgenix](https://bitbucket.org/gavinband/bgen/wiki/bgenix) indexer to generate a `.bgi` file for each `.bgen` file (for each chromosome). This feature will be added soon to ukbrest, so you don't have to download and compile bgenix.

Finally, run ukbrest with:

```bash
$ docker run --rm --net ukb -p 5000:5000 \
  -v /mnt/genotype:/var/lib/genotype \
  -e UKBREST_SQL_CHUNKSIZE="10000" \
  -e UKBREST_DB_URI="postgresql://test:test@pg:5432/ukb" \
  miltondp/ukbrest
```

Make sure the directory `/mnt/genotype` (you can choose another one) has you genotype data (chr21impv1.bgen, chr21impv1.bgen.bgi, etc).


## Querying genotype data

Query chromosome 1, positions from 0 to 1000:
```bash
$ curl -HAccept:application/octel-stream \
  "http://localhost:5000/ukbrest/api/v1.0/genotype/1/positions/0/1000" \
  > test.bgen
```

Query by chromosome and a file specifying rsids:
```bash
$ cat rsids.txt
rs367896724
rs540431307
rs555500075
rs548419688
rs568405545
rs534229142
rs537182016
rs376342519
rs558604819

$ curl -HAccept:application/octel-stream \
-X POST -F file=@rsids.txt \
"http://localhost:5000/ukbrest/api/v1.0/genotype/1/rsids" \
> test2.bgen
```


## Querying phenotype data

CSV format:
```bash
$ curl -HAccept:text/csv \
"http://localhost:5000/ukbrest/api/v1.0/phenotype\
?columns=c21_0_0\
&columns=c34_0_0\
&filters=eid <1000100"
```

Phenotype format (used by plink2, for example):
```bash
$ curl -HAccept:text/phenotype \
"http://localhost:5000/ukbrest/api/v1.0/phenotype\
?columns=c21_0_0\
&columns=c34_0_0\
&filters=eid <1000100"
```
