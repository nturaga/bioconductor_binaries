# Create Binaries for the AnVIL image

## Key points:

1. Make sure packages from /usr/local/lib/R/site-library are not used.

2. Make sure BiocBBSpack is not part of the binaries distributed. 

## TODO

1. Update image to R 4.0.2 

2. Test binary package creation and blob storage on Azure.

## Steps to create binaries

### Step 1: Investigate .libPaths()

Upon creating a new AnVIL run time based on
`us.gcr.io/anvil-gcr-public/anvil-rstudio-bioconductor:0.0.6`, check
for the following:

```
> sapply(.libPaths(), function(x) length(dir(x)))
       /home/rstudio/packages /usr/local/lib/R/site-library      /usr/local/lib/R/library 
                            0                           237                            30 
> existing_packages <- dir(.libPaths()[2])
```

There are 237 packages in `/user/local/lib/R/site-library` that are
being used to create the binary packages. These 237 packages will not
have binaries created for them.

They need to be deleted before moving forward with binary package
creation.

### Step 2: Delete packages in /usr/local/lib/R/site-library

Delete all packages in `.libPaths()[2]`.

```
> unlink("/usr/local/lib/R/site-library/", recursive = TRUE)

> sapply(.libPaths(), function(x) length(dir(x)))
       /home/rstudio/packages /usr/local/lib/R/site-library      /usr/local/lib/R/library 
                            0                             0                            30 
```

### Step 3: Install BiocManager

The primary location for installation of packages is in
`.libPaths()[1]` which is `/home/rstudio/packages`. So, the new
installation of BiocManager goes into `/home/rstudio/packages`.

```
install.packages('BiocManager')
```

### Step 4: Explicit install of BiocManager version

This is to make sure that the version of R matches that of
Bioconductor to ensure validity.

```
BiocManager::install(version = '3.11')
```

### Step 5: Install package 'remotes'

```
BiocManager::install('remotes')
```

### Step 6: Set options for Cpu usage across all installs

This document shows the Cpus as 60, but it can be figured out by
`parallel::detectCores() - 2`.

```
options(Ncpus = 60)
```

### Step 7:  Filter Vince's package from final list of installed packages.

Install Vince's package and get manifest `force=TRUE` is needed to
force installation, even if the remote state has not changed since the
previous install.

```
BiocManager::install("vjcitn/BiocBBSpack", force = TRUE)
```

**NOTE:** Right now as things stand with the image in `tidyr` fails
because it runs on R 4.0.0. This image needs to be udpated.

### Step 8: Get manifest and reinstall all packages

Make sure to reinstall packages which we deleted in Step 2 from
`/usr/local/lib/R/site-library`. 

```
pl = BiocBBSpack::get_bioc_packagelist("master")

pl = union(pl, existing_packages)

BiocManager::install(pl, force=TRUE)
```

### Step 9: View packages in /home/rstudio/packages

Create a new location for binary packages. 

```
.libPaths()[1]

## Create dir
dir.create("/home/rstudio/binaries")
```

### Step 10: Create tar files

```
setwd("/home/rstudio/packages")
jnk = lapply(dir(), BiocBBSpack:::dotarmv, dest = "/home/rstudio/binaries")
```

**NOTE: Remove BiocBBSPack from /home/rstudio/binaries**. This is
needed so that the binaries are only Bioconductor packages and
dependencies

```
file.remove("/home/rstudio/binaries/BiocBBSpack_0.0.50.tar.gz")
```

### Step 11: Create PACKAGES.gz

TODO: FIX this

```
tools::write_PACKAGES(dir = "/home/rstudio/packages/", unpacked=TRUE)

## Move it to /home/rstudio/binaries
file.rename("/home/rstudio/packages/PACKAGES.gz", "/home/rstudio/binaries/PACKAGES.gz")
```

### Step 12: Move to bucket on Gcloud

Authenticate with Google Cloud on RStudio terminal

```
## Follow instructions after this command and verify the project 
gcloud auth login
```

Make sure the project is set correctly,

```
gcloud config set project bioconductor-rpci-280116
```

Create a google bucket in a CRAN style manner, with `gsutil mb`. This
bucket will be called `anvil-rstudio-bioconductor-test` since it's a
**GLOBALLY** unique name. 

```
gsutil mb gs://anvil-rstudio-bioconductor-test/0.99/3.11/src/contrib/

## Check if the bucket is created under your account

gsutil ls
```

### Step 13: transfer all the contents over

```
## gsutil copy everythin in /home/rstudio/binaries over
gsutil -m \
	cp -r \
	/home/rstudio/binaries/* \
	gs://anvil-rstudio-bioconductor-test/0.99/3.11/src/contrib/
```

### Step 14: Make bucket public

The bucket needs to be public to generate the HTTPS interface to the public.

```
gsutil iam ch allUsers:objectViewer gs://anvil-rstudio-bioconductor-test/
```

### Step 15: Test is packages are installable as binaries

The public access point through HTTPS is given with the link
`https://storage.googleapis.com`. So the bucket we created looks like:

- https://storage.googleapis.com/anvil-rstudio-bioconductor-test/0.99/3.11/src/contrib/


```
repo <- "https://storage.googleapis.com/anvil-rstudio-bioconductor-test/0.99/3.11/"

## This should install the package from the binary location
BiocManager::install('ABarray', site_repository = repo)
```
