# Yuvi's Guide to Building JupyterHub images

<a rel="license" href="http://creativecommons.org/licenses/by/4.0/"><img
alt="Creative Commons License" style="border-width:0"
src="https://i.creativecommons.org/l/by/4.0/88x31.png" /></a>This work is
licensed under a <a rel="license"
href="http://creativecommons.org/licenses/by/4.0/">Creative Commons Attribution
4.0 International License</a>.

Welcome to my first ever 'book' of any sort :) This is an attempt to extract knowledge that
seems to be stuck in my head on building Docker images of various sorts for use with JupyterHubs.
These are somewhat opinionated, but have worked well across various communities & institutions for
many years now. Even if you do not want to adopt any of it wholesale, I hope there are enough 
parts you can pick and choose to use from here.

While the focus of this book is on building images for use with JupyterHub, everything here applies
100% to building images that work on [mybinder.org](https://mybinder.org) or any other BinderHubs.

## Scope

The goal of this book is two-fold:

1. Help you pick the method of docker image building that is *right for you*,
   without requiring you to be an expert in containers.
2. Provide detailed how-to guides on using these methods to build *and maintain*
   images for interactive datascience use.
   
If you are already an expert in container technology, and the images you build
will be maintained by others who are also experts in container technology,
maybe this book is not for you.

## Methods covered in this book

The 4 methods I want to cover in this book are:

1. Using [repo2docker](https://github.com/jupyterhub/repo2docker)
2. Using an unmodified community curated image, such as
   [jupyter docker-stacks](https://jupyter-docker-stacks.readthedocs.io/en/latest/), 
   [pangeo images](https://github.com/pangeo-data/pangeo-docker-images) or 
   the [rocker images](https://rocker-project.org/).
3. Inheriting from and modifying a community curated base image (an extension of (2))
4. Building your own image from scratch, with a `Dockerfile`.

As with life, there are trade-offs in making this choice. This book's primary
objective is to help you make this choice, and follow-through with it.
