# Unicity BFT Core Documentation

## Pre-built PDF:

(unicity-bft-core.pdf)[unicity-bft-core.pdf]

## Building locally (using docker):

- Pull the image:

    `docker pull registry.gitlab.com/islandoftex/images/texlive:latest` (might need to put `sudo` in front)

- Build the paper (from the project root): note that `pdflatex` has to be executed 3 times (to resolve all cross-references)

    `docker run -i --rm --name latex -v "$PWD":/usr/src/app -w /usr/src/app/ registry.gitlab.com/islandoftex/images/texlive:latest pdflatex unicity-bft-core.tex`

## Building locally:

1. install a complete tex environment

    `brew install mactex-no-gui`  (macOS, using homebrew)

    `apt-get install texlive-full` (debian and derivatives)

2. Build using three passes of `pdflatex` or use the provided Makefile.
