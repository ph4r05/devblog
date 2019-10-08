---
layout: post
title:  "Jupyter notebook remotely"
date:   2019-10-08 13:00:00 +0200
categories: blog
excerpt_separator: <!-- more -->

---

Running jupyter notebook remotely on a server without root privileges

<!-- more -->

## Intro

Motivation for this post is having running Jupyter notebook long-term on a server which has more resources than your workstation
and can compute, process data, communicate over the Internet even when your workstation is off. 

The post explains how to deploy Jupyter notebook on a server with user privileges. We focus on our faculty environment
which runs RHEL 8 (servers called Aisa and Aura) with loadable modules support. However, the deployment procedure can be generalized to any server.

### Outline

- [pyenv] installation
- python packages installation
- running & connecting to a remote jupyter notebook  

## Installation

As we don't have root privileges we need to install custom Python version so we can use `pip` to install packages as we need.
We use [pyenv] for this purpose, which enables us to build and use custom python versions.

Requirements:
- `openssl 1.1.1`
- `zlib`
- `libffi`
- `bzip2` 
- `lzma`     (optional)
- `readline` (optional)

If the requirements are not met, ask administrator to install them or build them from source and install locally to your home folder (`./configure --prefix=$HOME/local && make && make install`, then adjust environment variables).

To build the Pyenv on Aisa/Aura servers you can use module system to add all required dependencies:

```bash
module load openssl-1.1.1  zlib-1.2.11  readline-8.0 libffi-3.2.1 bzip2 \
       lzma-4.32.7 sqlite-3.29.0 mariadb-client-8.0.17 graphviz-2.26.3 
```

### Pyenv installation

[pyenv] installation steps are taken from the project page:

```bash
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bashrc
exec "$SHELL"
```

### Pyenv build

Now you need to build the Python version, we will use `3.7.1`

```bash
pyenv install -v 3.7.1
pyenv global 3.7.1
```

To compile on Aura we need to adjust environment variables so compiler can find loaded modules:

```bash
MAKEFLAGS=-j26 \
CFLAGS="-I/packages/run.64/openssl-1.1.1/include -I/packages/run.64/libffi-3.2.1/lib/libffi-3.2.1/include -I/packages/share/readline-8.0/include -I/packages/share/zlib-1.2.11/include -I/packages/share/sqlite-3.29.0/include -I/packages/run.64/bzip2-1.0.2/include -I/packages/share/lzma-4.32.7/include" \
LDFLAGS="-L/packages/run.64/openssl-1.1.1/lib/ -L/packages/run.64/libffi-3.2.1/lib64 -L/packages/run.64/readline-8.0/lib -L/packages/run.64/zlib-1.2.11/lib -L/packages/run.64/sqlite-3.29.0/lib -L/packages/run.64/lzma-4.32.7/lib -L/packages/run.64/bzip2-1.0.2/lib"  \
CPPFLAGS="-I/packages/run.64/openssl-1.1.1/include -I/packages/run.64/libffi-3.2.1/lib/libffi-3.2.1/include -I/packages/share/readline-8.0/include -I/packages/share/zlib-1.2.11/include -I/packages/share/sqlite-3.29.0/include -I/packages/run.64/bzip2-1.0.2/include -I/packages/share/lzma-4.32.7/include"  \
CONFIGURE_OPTS="--with-openssl=/packages/run.64/openssl-1.1.1"  \
LD_RUN_PATH=$LIBRARY_PATH \
pyenv install -v 3.7.1
pyenv global 3.7.1
```

Now you can install additional packages:

```bash 
pip install jupyter pandas matplotlib numpy networkx graphviz scipy coloredlogs mysqlclient \
    requests sarge cryptography paramiko shellescape
```

## Running Jupyter

In order to run Jupyter notebook on the server you need to start it in a `screen` or `tmux` so the process runs in the background and survives SSH session termination.

Then you need to specify a port which will Jupyter use for incoming connections. If there are several users sharing the server for the same purpose of Jupyter notebook sharing you should use unique port number.

Create following file `jupyter.sh` on the server:

```bash 
#!/bin/bash
# jupyter notebook password
# jupyter notebook list
: "${JPORT:=8876}"  # make unique per user
module load openssl-1.1.1  zlib-1.2.11  readline-8.0 libffi-3.2.1 bzip2 lzma-4.32.7 sqlite-3.29.0 mariadb-client-8.0.17 graphviz-2.26.3 
jupyter-notebook  --no-browser --port $JPORT .
```

Then run the script `jupyter.sh` in the `screen`.

```bash 
screen -mS jupyter  # creates a new screen named jupyter
./jupyter.sh

# Copy the login link returned by the jupyter - you will need it to login to notebook later.
# CTRL + A + D to detach the screen, keeps jupyter running
```

Now is jupyter running on a server loopback interface, namely `127.0.0.1:$JPORT`.
In order to connect to the notebook via browser you need to establish a SSH tunnel to the server:

```bash 
: "${JPORT:=8876}"
ssh -L $JPORT:localhost:$JPORT aura -t 'jupyter notebook list; bash -l'
```

This command opens a SSH connection and creates a tunnel to the jupyter notebook socket running on the server. 
Now you can visit `http://127.0.0.1:$JPORT` locally, in your web browser. However, to login to the notebook you need to use login link provided by the jupyter notebook after start on the server.


[pyenv]: https://github.com/pyenv/pyenv
