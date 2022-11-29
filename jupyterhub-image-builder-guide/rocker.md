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

So let's build the most basic `Dockerfile` to get started!

```dockerfile
FROM rocker/geospatial:4.2.2

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

# Install latest mambaforge in ${CONDA_DIR}
RUN echo "Installing Mambaforge..." \
    && URL="https://github.com/conda-forge/miniforge/releases/latest/download/Mambaforge-Linux-x86_64.sh" \
    && wget --quiet ${URL} -O installer.sh \
    && /bin/bash installer.sh -u -b -p ${CONDA_DIR} \
    && rm installer.sh \
    && mamba clean -afy \
    # After installing the packages, we cleanup some unnecessary files
    # to try reduce image size - see https://jcristharif.com/conda-docker-tips.html
    && find ${CONDA_DIR} -follow -type f -name '*.a' -delete \
    && find ${CONDA_DIR} -follow -type f -name '*.pyc' -delete
```

Let's explain what exactly is going on here.

### The Base Image 

The `FROM` line specifies which docker image and version we want to use.

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
declared via `ENV` upto that point, and so can expand them. So our `RUN echo "PATH=${PATH}"` gets translated
into `RUN ["/bin/sh", "-c", "echo \"PATH=${PATH}\"]`, and `sh` expands the `${PATH}` to the full value of
the environment variable as they are enclosed in double quotes. This allows us to tell RStudio Server to
set its `PATH` environment variable to the exact value of the `PATH` environment variable in our Dockerfile,
in the most roundabout way possible :)

RStudio Server has a terminal, and that also does not inherit environment variables set here with `ENV` - and
since it isn't an R process, neither does it inherit environment variables set in `Renviron.site`. So instead
we put it into `/etc/profile`, which is loaded by the Terminal inside RStudio since it starts `bash` with the
`-l` argument - which causes it to read `/etc/profile`! We use the same variable expansion trick as we did
in the previous step as well.
   
Aren't computers wonderful? :)


