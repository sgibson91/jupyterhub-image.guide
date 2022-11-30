# R images based on [the Rocker Project](https://rocker-project.org/)

If your JupyterHub users are going to be *primarily* using R, we recommend
basing your user image on one of the images maintained by the [Rocker Project](https://rocker-project.org/)

## Why?

The Rocker Project provides a collection of [different images](https://rocker-project.org/images/)
that come with popular R software and optimized libraries pre-installed. It is maintained by people
deeply vested in the R community, and provides curated sets of base libraries with versions known to
work decently well together.

%TODO: Sing praises of Rocker some more

## Step 1: Pick a base image

While rocker offers a bunch of [different images](https://rocker-project.org/images/) to pick from, my recommendation is
to base your image on the [rocker/geospatial](https://hub.docker.com/r/rocker/geospatial/tags) image.
It isn't particularly large, and offers a fairly complete & comprehensive set of
base packages to build off of.

```{note}

If you particularly care about shaving off image size, you could also consider using the
[rocker/verse](https://hub.docker.com/r/rocker/verse/tags) image. At the time of writing, it is only about
a 300MB reduction in size - which in not much in the context of datascience images.
```

## Step 2: Pick a version

All the rocker images, including `rocker/geospatial`, are tagged based on the R version they use.
Look in the [tags page](https://hub.docker.com/r/rocker/geospatial/tags) for the base image you have picked,
and find the tag that corresponds to the R version you wanna use. In general, R users seem to upgrade R versions
fairly quickly after a new release - so it's a decent bet to just pick the latest released version in the
tag list. Pick a fully specified version - so `4.2.2`, rather than `4.2` or `4`. This lets you keep the R version
and base set of packages stable until explicitly bumped. You should bump the base version at least once every 6 months.

So at this point, the `Dockerfile` looks like this:

```dockerfile
FROM rocker/geospatial:4.2.2
```

%TODO: Figure out what happens to package versions?!

## Step 3: Construct your Dockerfile to add Python

We need to install Python inside the user image, even if our JupyterHub users will not be using
any Python. This is for the following reasons:

1. The `jupyterhub-singleuser` executable must be available inside the container image for the image
   to be used with a JupyterHub.
2. The [jupyter-rsession-proxy](https://github.com/jupyterhub/jupyter-rsession-proxy) is required to get
   RStudio to work with JupyterHub.
   
While there are many different ways to install Python inside a container image, we choose to use   
[mamba](https://mamba.readthedocs.io/en/latest/) - a popular drop-in replacement for the [conda](https://docs.conda.io/en/latest/)
package manager. Why?

1. It is how [repo2docker](https://repo2docker.readthedocs.io/en/latest/) and hence [mybinder.org](https://mybinder.org)
   get their Python, so the consistency is very helpful.
2. We can get *any* version of python we want, not just what is supported by the underlying Linux
   distro. While there are other ways to get arbitrary Python versions, in my experience this has been
   the easiest and most stable way.

So let's add on to our `Dockerfile`!

```dockerfile
# Install conda here, to match what repo2docker does
ENV CONDA_DIR=/srv/conda

# Add our conda environment to PATH, so python, mamba and other tools are found in $PATH
ENV PATH ${CONDA_DIR}/bin:${PATH}

# RStudio doesn't actually inherit the ENV set in Dockerfiles, so we
# have to explicitly set it in Renviron.site
RUN echo "PATH=${PATH}" >> /usr/local/lib/R/etc/Renviron.site

# The terminal inside RStudio doesn't read from Renviron.site, but does read
# from /etc/profile - so we rexport here.
RUN echo "export PATH=${PATH}" >> /etc/profile

# Install a specific version of mambaforge in ${CONDA_DIR}
# Pick latest version from https://github.com/conda-forge/miniforge/releases
ENV MAMBAFORGE_VERSION=22.9.0-2
RUN echo "Installing Mambaforge..." \
    && curl -sSL "https://github.com/conda-forge/miniforge/releases/download/${MAMBAFORGE_VERSION}/Mambaforge-${MAMBAFORGE_VERSION}-Linux-$(uname -m).sh" > installer.sh \
    && /bin/bash installer.sh -u -b -p ${CONDA_DIR} \
    && rm installer.sh \
    && mamba clean -afy \
    # After installing the packages, we cleanup some unnecessary files
    # to try reduce image size - see https://jcristharif.com/conda-docker-tips.html
    && find ${CONDA_DIR} -follow -type f -name '*.a' -delete \
    && find ${CONDA_DIR} -follow -type f -name '*.pyc' -delete
```

Let's explain what exactly is going on here.

### Setting location of the conda environment

We set a `CONDA_DIR` environment variable with the path to where we want to
install the conda environment. While many conda installs are put in
`/opt/conda`, we choose to put it in `/srv/conda` still to match what
`repo2docker` does.

```{note}
Why is it called CONDA_DIR while we are installing mamba? Backwards compatibility, and the fact that we are
still using the conda *ecosystem* heavily - particularly [conda-forge](https://conda-forge.org/).
```
   
%TODO: Clarify `mamba` vs `conda environment` vs `conda package` vs `conda-forge`


### Making tools in the conda environment available on `${PATH}`

We set the `PATH` environment variable to include `${CONDA_DIR}/bin`, so users can invoke various software
tools (like `python`) we install via `mamba` without having to specify the full path. This is also required
for JupyterHub to use the image - we will install `jupyterhub-singleuser` into this conda environment later,
and adding this directory to `PATH` allows jupyterhub to start `jupyterhub-singleuser` without having to 
know where *exactly* it is installed.

#### Special considerations for environment variables inside RStudio
   
The Dockerfile [ENV](https://docs.docker.com/engine/reference/builder/#env) directive sets up environment
variables inherited by all the processes started by JupyterHub inside the container. **However**, RStudio
Server does *not* inherit these! This is mostly due to how RStudio Server starts - by daemonizing itself
using the technique of the ancient powerful ones, [double forking](https://stackoverflow.com/questions/881388/what-is-the-reason-for-performing-a-double-fork-when-creating-a-daemon).
   
So for env variables to take effect in the R sessions started by RStudio, they need to be added to an [`Renviron.site`](https://support.posit.co/hc/en-us/articles/360047157094-Managing-R-with-Rprofile-Renviron-Rprofile-site-Renviron-site-rsession-conf-and-repos-conf). Add one line per file with the value of the
environment variable you want, and it shall be loaded by R on startup when started by RStudio. This way, if
you have code in R that looks for a python (or other) executable, it will find it in the correct place.
   
But wait, what does `PATH=${PATH}` do?! How does *that* work? If you look at the documentation for the
Dockerfule [RUN](https://docs.docker.com/engine/reference/builder/#run) directive, `RUN echo HI` is actually
equivalent to `RUN ["/bin/sh", "-c", "echo HI"]`! And this `/bin/sh` *does* get all the environment variables
declared via `ENV` upto that point, and so can [expand
them](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html).
So our `RUN echo "PATH=${PATH}"` gets translated into `RUN ["/bin/sh", "-c",
"echo \"PATH=${PATH}\"]`, and `sh` expands the `${PATH}` to the full value of
the environment variable as they are enclosed in double quotes. This allows us
to tell RStudio Server to set its `PATH` environment variable to the exact value
of the `PATH` environment variable in our Dockerfile, in the most roundabout way
possible :)

RStudio Server has a terminal, and that also does not inherit environment variables set here with `ENV` - and
since it isn't an R process, neither does it inherit environment variables set in `Renviron.site`. So instead
we put it into `/etc/profile`, which is loaded by the Terminal inside RStudio since it starts `bash` with the
`-l` argument - which causes it to read `/etc/profile`! We use the same variable expansion trick as we did
in the previous step as well.
   
Aren't computers wonderful? :)

### Installing Mambaforge

We then use the [Mambaforge Installer](https://github.com/conda-forge/miniforge#mambaforge) to install
create a conda environment at `$CONDA_DIR`. What exactly does "Mambaforge" give us?

1. A conda environment set to install packages from the community maintained [conda-forge](https://conda-forge.org/)
   channel, filled with thousands of amazingly well maintained packages in python & other languages.
2. The [mamba](https://mamba.readthedocs.io/en/latest/) package manager by default, the faster drop-in
   replacement to the conda package manager.
3. A *default version of Python*, based on which *version* of mambaforge we picked. Usually the version that
   comes here is mentioned when you pick the release from the [releases page]( https://github.com/conda-forge/miniforge/releases)

Notice that bit of `$(uname -m)` in there? It automatically picks either the intel CPU installer (x86_64) or the
ARM CPU Installer (aarch64) based on what architecture we are building our image for *automatically*. This might
not matter to you now, but ARM machines are taking over the world at an exponentially increasing rate (I am writing
this on an ARM machine right now) - and by adding support for ARM from the start, we future-proof our image
without a lot of extra work!

Since the mambaforge installer was originally designed to be used on people's laptops, it does leave behind
some cached files and what not that are of not much use in a docker container - after all, we do not expect
our end users to constantl re-install packages inside the container while they are using it (for the most part),
so we can save some disk space by cleaning these up before we finish the `RUN` directive. Note that these all
need to be part of the same command, otherwise there will be no space saving! See [this wonderful blog post]( https://jcristharif.com/conda-docker-tips.html)
for more details.

## Step 4: Installing Python Packages with an `environment.yml` file

Now that we have *python* installed, it's time to install a few python packages in the conda
environment.

First, we create an `environment.yml` file listing the packages we want installed. This is a [standard
format](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#create-env-file-manually)
used in the conda ecosystem.

```yaml
channels:
- conda-forge
- nodefaults

dependencies:
- jupyterhub-singleuser>=3.0
- jupyter-rsession-proxy>=2
```

This is the absolute *minimal* `environment.yml` file you need. Let's unpack what it has!

### `channels`

conda packages are organized into [channels](https://docs.conda.io/projects/conda/en/latest/user-guide/concepts/channels.html),
which are locations where packages are stored. The most common ones are:

1. The [conda-forge](https://anaconda.org/conda-forge) channel, containing packages built by the massive
   [conda-forge](https://conda-forge.org/) community. This is where we *want* most of our packages to come
   from.
2. The "defaults" channel, which contains packages managed by [Anaconda.com](https://www.anaconda.com/),
   primarily for use with their [Anaconda Python Distribution](https://www.anaconda.com/products/distribution).
   This is **always** present by default, but since packages present in this are also usually present in the
   conda-forge channel, accidental mixing can be confusing and problematic.
3. The special "nodefaults" channel, which removes the "defaults" channel from being considered for package
   installation! You should always include this when building container images
4. The [nvidia](https://anaconda.org/nvidia/) channel, containing a mix of proprietary & open source packages
   for use with NVidia's CUDA ecosystem. You don't need this unless you are building stuff that uses a GPU.
   
Specifying `conda-forge` and `nodefaults` is most likely all you will need!

%TODO: Move this out somewhere else and crosslink?

### `dependencies`

This lists the conda packages we want to install in our environment. The two
packages we *must* add at a minimum are
[jupyterhub-singleuser](https://anaconda.org/conda-forge/jupyterhub-singleuser)
(required for any image to work with JupyterHub), and
[jupyter-rsession-proxy](https://anaconda.org/conda-forge/jupyter-rsession-proxy)
(provides working RStudio Server support within JupyterHub). You can add additional packages here if
you wish.

```{note}
We explicitly chose to *not* specify a python version here, so we just use the version of python
that mambaforge installs by default. This keeps the image size small! If you want a newer version of
python, I recommend changing the version of Mambaforge you are installing
```

### Installing the packages into the existing environment

Now that we have the `environment.yml` file, we can add it to our `Dockerfile` and install those packages
into the environment.


```dockerfile
COPY environment.yml /tmp/environment.yml

RUN mamba env update -p ${CONDA_DIR} -f /tmp/environment.yml \
    && mamba clean -afy \
    && find ${CONDA_DIR} -follow -type f -name '*.a' -delete \
    && find ${CONDA_DIR} -follow -type f -name '*.pyc' -delete
```

We also cleanup after ourselves to keep the image size to a minimum - we will
have to do this after every `mamba` command for the most part!

## Step 5: Install additional R Packages

While the base rocker image comes with a *lot* of useful libraries, you might still need to
add more. While there are many ways to install R packages, I recommend using the
[install2.r](https://rocker-project.org/use/extending.html#install2.r) command that comes
with rocker by default (via the [littler](https://github.com/eddelbuettel/littler) project).

By default, packages will be installed from [packagemanager.rstudio.com](https://packagemanager.rstudio.com/client/),
which provides optimized binary installs that install quite quickly!

```dockerfile
RUN install2.r --skipinstalled \
    package1 \
    package2 \
    && rm -rf /tmp/downloaded_packages
```

This will fetch appropriate versions of `package1`, `package2`, etc and install them *only if they are not
already installed*. If you don't specify `--skipinstalled`, packages from the base image might keep getting
reinstalled - wasting time and space. We cleanup after ourselves by deleting temporary directories used for
storing downloaded packages before installation.

%TODO: Figure out version pins?!

## (Optional) Step 6: Install Jupyter Notebook with R support

While most R users prefer using RStudio, there are are also many users of the Jupyter [IRkernel](https://irkernel.github.io/)
for using R with Jupyter Notebooks. It is generally a good idea to support this too in images we build for
R users.

### Install Jupyter frontends (JupyterLab & RetroLab)

First, we install a couple of Jupyter frontends that people can use to edit their notebooks by adding
them under `dependencies` to the `environment.yml`. Without this, there won't be any Jupyter frontends
for people to use for reading / writing Jupyter Notebooks!

I recommend installing the
[`jupyterlab`](https://github.com/jupyterlab/jupyterlab/) and
[`retrolab`](https://github.com/jupyterlab/retrolab) frontends. The former provides an IDE-like experience,
while the latter provides a much simpler single-document experience!

With these added, the `environment.yml` would look something like:

```yaml
channels:
- conda-forge
- nodefaults

dependencies:
- jupyterhub-singleuser>=3.0
- jupyter-rsession-proxy>=2
- jupyterlab>=3.0
- retrolab
```

### Install the R Kernel for Jupyter (`IRkernel`)

Now that Jupyter frontends are installed, we need to install the [Jupyter R Kernel](https://irkernel.github.io/).
Without this, only the default Python kernel would be available.

```dockerfile
RUN install2.r --skipinstalled IRkernel \
    && rm -rf /tmp/downloaded_packages
    
RUN r -e "IRkernel::installspec(prefix='${CONDA_DIR}')"
```

We first install R package that provides the kernel, and then tell it to 'install' itself as an available
kernel for the Jupyter installation we have. 

### Tell Jupyter where to start

The rocker images come with a default non-root user named `rstudio`, with a writeable home directory at `/home/rstudio`.
While RStudio Server knows to start in `/home/rstudio`, Jupyter does not. We have to tell it explicitly to
start  in `/home/rstudio`, otherwise it will start in `/` and users will not be able to create notebooks there!

```dockerfile
# Explicitly specify working directory, so Jupyter knows where to start
WORKDIR /home/rstudio
```

### Tell Jupyter to use a better shell when starting Terminals

Jupyter also has the capability to launch terminals, but will default to launching `/bin/sh` as the terminal
shell. This is not quite right, as most people expect instead for `/bin/bash` to be launched - so features like
autocomplete and arrow keys work! We have to explicilty tell Jupyter to launch `/bin/bash` by setting the `SHELL`
environment variable.

```dockerfile
ENV SHELL=/bin/bash
```

```{note}
RStudio Server doesn't need this to launch `/bin/bash`, so we don't need to repeat the tricks we used in Step 3
to set env vars for RStudio Server here.
```

## Step 8: Setup zero to JupyterHub configuration for home directory

When using [zero-to-jupyterhub-on-k8s](https://z2jh.jupyter.org) for running JupyterHub, the *default* persistent home directory for the
user is mounted at `/home/jovyan`. This works for *most* cases, as the default user name for most container images
used with JupyterHub is `jovyan`, and their home directories are set to `/home/jovyan`. **However**, since rocker's
default username is `rstudio` and the home directory is set to `/home/rstudio`, we must explicitly tell z2jh to
mount the user's persistent home directory there, with the following config:

```yaml
singleuser:
  storage:
    homeMountPath: /home/rstudio
```

```{warning}
Without setting this, the user's persistent home directory will continue to be mounted at `/home/jovyan`, **but**
the user will only see `/home/rstudio` by default. **This results in data loss**, as `/home/rstudio`'s contents
are discarded when the user's server stops! You **do not want this**, so remember to set this configuration!
```
