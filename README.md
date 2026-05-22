## Seqera Studio R-IDE

This repository contains the definition of Seqera Studio R-IDE. The `main` branch contains the
latest supported version of R-IDE with connect-client. To use an older version of any component,
check the existing tags using the pattern: `r-ide/<version>/connect/<version>`.

## Components

- **[RStudio Server](https://posit.co/products/open-source/rstudio-server/) (Seqera R-IDE v2025.05.0+496)** — browser-based R development environment
- **R 4.4.1** — via micromamba/conda-forge
- **Python 3.13** — via micromamba/conda-forge
- **R packages** — r-markdown, pandoc
- **NGINX** — reverse proxy for path-based routing
- **connect-client** — Seqera connect client for studio integration

## Repository Structure

`.seqera/` contains:

- `studio-config.yaml` — references the pre-built image; studios using this branch will not require a build step
- `Dockerfile` — shows how the image was built; fork this repository and modify it to create a custom image
- `env.yaml` — conda environment specification (Python 3.13, R 4.4.1, r-markdown, pandoc)
- `init` — startup script that launches RStudio Server and NGINX
- `rserver.conf` — RStudio Server configuration
- `rsession.conf` — RStudio session configuration
- `logging.conf` — RStudio logging configuration
- `nginx.conf.template` — NGINX reverse proxy template for path-based routing
- `Renviron` — R environment configuration
- `keybindings/` — RStudio keybinding configurations

## Customization

To create a customized version:

1. Fork this repository
2. Modify the `Dockerfile` and/or `env.yaml` to add your tools or dependencies
3. Build and push your custom image
4. Update `studio-config.yaml` to reference your custom image

## Pre-built Image

The pre-built image is available at:

```
public.cr.seqera.io/platform/data-studio-ride:2026.01.2-0.12.0
```
