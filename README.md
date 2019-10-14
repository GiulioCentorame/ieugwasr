# R interface to the IEU GWAS database API

<!-- badges: start -->
<!-- badges: end -->

R interface to the IEU GWAS database API. Includes a wrapper to make generic calls to the API, plus convenience functions for specific queries.

## Installation

You can install the released version of ieugwasr from [CRAN](https://CRAN.R-project.org) with:

``` r
devtools::install_github("mricue/ieugwasr")
```

## Usage

### General API queries

The API has a number of endpoints documented [here](http://ieu-db-interface.epi.bris.ac.uk:8082/docs/). A general way to access them in R is using the `api_query` function. There are two types of endpoints - `GET` and `POST`. 

- `GET` - you provide a single URL which includes the endpoint and query. For example, for the `association` endpoint you can obtain some rsids in some studies, e.g.
    + `api_query("associations/IEU-a-2,IEU-a-7/rs234,rs123")`

- `POST` - Here you send a "payload" to the endpoint. So, the path specifies the endpoint and you add a list of query specifications. This is useful for long lists of rsids being queried, for example
    + `api_query("associations", query=list(rsid=c("rs234", "rs123"), id=c("IEU-a-2", "IEU-a-7")))`

### Authentication

Most datasets in the database are public and don't need authentication. But if you want to access a private dataset that is linked to your (gmail) email address, you need to authenticate the query using a method known as Google OAuth2.0.

Essentially - you run this command at the start of your session: 

```
get_access_token()
```

which will open up a web browser asking you to provide your google username and password, and upon doing so a directory will be created in your working directory called `ieugwasr_oauth`. This directory contains a file that looks like this: `<random_string>_<email@address>`. It is a binary file (not human readable), which contains your access token, and it acts as a convenient way to hold a randomly generated password.

If you are using a server which doesn’t have a graphic user interface then the `get_access_token()` method is not going to work. You need to generate the `ieugwasr_oauth` directory and token file on a computer that has a web browser, and then copy that directory (containing the token file) to your server (to the relevant work directory).

If you are using R in a working directory that does not have write permissions then this command will fail, please navigate to a directory that does have write permissions.

If you need to run this in a non-interactive script then you can generate the token file on an interactive computer, copy that file to the working directory that R will be running from, and then run a batch (non-interactive).


### Convenient wrappers for `api_query`

It can be cumbersome to generate the query manually, so here are some convenient functions to run various operations

### Get API status

```r
api_status()
```

### Get list of all available studies

```r
gwasinfo()
```

### Get list of a specific study

```r
gwasinfo("IEU-a-2")
```

### Extract particular associations from particular studies

Provide a list of variants to be obtained from a list of studies:

```r
associations(variants=c("rs123", "7:105561135"), id=c("IEU-a-2", "IEU-a-7"))
```

By default this will look for LD proxies using 1000 genomes reference data (Europeans only, the reference panel has INDELs removed and only retains SNPs with MAF > 0.01). This behaviour can be turned off using `proxies=0` as an argument.

Note that the queries are performed on rsids, but chromosome:position values will be automatically converted. A range query can be done using e.g.

```r
associations(variants="7:105561135-105563135", id=c("IEU-a-2"), proxies=0)
```


### Get tophits from study

The tophits can be obtained using

```
tophits(id="IEU-a-2")
```

Note that it will perform strict clumping by default (r2 = 0.001 and radius = 10000kb). This can be turned off with `clump=0`.


### Perform PheWAS

(Under development).

### LD clumping

The API has a wrapper around [plink version 1.90](https://www.cog-genomics.org/plink/1.9) and can use it to perform clumping with an LD reference panel from 1000 genomes reference data (Europeans only, the reference panel has INDELs removed and only retains SNPs with MAF > 0.01).

```
a <- tophits(id="IEU-a-2", clump=0)
b <- ld_clump(
    dplyr::tibble(variant=a$name, pval=a$p, id=a$id)
)
```

Note that you can perform the same operation locally if you provide a path to plink and a bed/bim/fam LD reference dataset. e.g.

```
ld_clump(
    dplyr::tibble(variant=a$name, pval=a$p, id=a$id),
    plink_bin = "/path/to/plink",
    bfile = "/path/to/reference_data"
)
```

### LD matrix

Similarly, a matrix of LD r values can be generated using

```
ld_matrix(b$variant)
```

This uses the API by default but is limited to only 500 variants. You can use, instead, local plink and LD reference data in the same manner as in the `ld_clump` function, e.g.

```
ld_matrix(b$variant, plink_bin = "/path/to/plink", bfile = "/path/to/reference_data")
```


### Variant information

Translating between rsids and chromosome:position, while also getting other information, can be achieved. 

The `chrpos` argument can accept the following

- <chr>:<position>
- <chr>:<start>-<end>

For example

```
a <- variants_chrpos(c("7:105561135-105563135", "10:44865737"))
```

This provides a table with dbSNP variant IDs, gene info, and various other metadata. Similar data can be obtained from searching by rsid

```
b <- variants_rsid(c("rs234", "rs333"))
```

And a list of variants within a particular gene region can also be found. Provide a ensembl or entrez gene ID (e.g. ENSG00000123374 or 1017) to the following:

```
c <- variants_gene("ENSG00000123374")
```

