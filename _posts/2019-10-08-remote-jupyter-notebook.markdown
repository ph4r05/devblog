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

Jupyter notebook is a great tool for fast code prototyping as it holds the state across the whole session so you can load large dataset in one cell and experiment with different processing techniques in different cells without need to reload the input data. Moreover, it nicely integrates with matplotlib/pandas/seaborn so all charts are displayed nicely in the web interface after rendering. 

[![Jupyter notebook example](/static/jupyter/jupyter-demo.png)](/static/jupyter/jupyter-demo.png) 

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

### 1. Requirements
- `openssl 1.1.1b`
- `zlib`
- `libffi`
- `bzip2` 
- `xz`       (optional)
- `readline` (optional)

If the requirements are not met, ask administrator to install them or build them from source and install locally to your home folder (`./configure --prefix=$HOME/local && make && make install`, then adjust environment variables).

Unless said otherwise, run all commands on the server.

To build the Pyenv on Aisa/Aura servers you can use module system to add all required dependencies:

```bash
module load zlib-1.2.11 ncurses-5.9 readline-7.0 libffi-3.2.1 bzip2-1.0.8 \
       xz-5.2.4 sqlite-3.29.0 mariadb-client-8.0.17 graphviz-2.26.3 
```

### 2. Pyenv installation

[pyenv] installation steps are taken from the project page:

```bash
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bashrc
exec "$SHELL"
```

### 3. Pyenv build

Now you need to build the Python version, we will use `3.7.3`

```bash
pyenv install -v 3.7.3
pyenv global 3.7.3
```

To compile on Aura we need to adjust environment variables so compiler can find loaded modules:

```bash
MAKEFLAGS=-j42 \
CFLAGS="-I/packages/run.64/libffi-3.2.1/lib/libffi-3.2.1/include \
        -I/packages/share/readline-7.0/include \
        -I/packages/share/zlib-1.2.11/include \
        -I/packages/share/sqlite-3.29.0/include \
        -I/packages/share/bzip2-1.0.8/include \
        -I/packages/share/ncurses-5.9/include \
        -I/packages/share/xz-5.2.4/include" \
LDFLAGS="-L/packages/run.64/libffi-3.2.1/lib64 \
        -L/packages/run.64/readline-7.0/lib \
        -L/packages/run.64/zlib-1.2.11/lib \
        -L/packages/run.64/sqlite-3.29.0/lib \
        -L/packages/run.64/xz-5.2.4/lib \
        -L/packages/run.64/ncurses-5.9/lib \
        -L/packages/run.64/bzip2-1.0.8/lib"  \
CPPFLAGS="-I/packages/run.64/libffi-3.2.1/lib/libffi-3.2.1/include \
        -I/packages/share/readline-7.0/include \
        -I/packages/share/zlib-1.2.11/include \
        -I/packages/share/sqlite-3.29.0/include \
        -I/packages/share/bzip2-1.0.8/include \
        -I/packages/share/ncurses-5.9/include \
        -I/packages/share/xz-5.2.4/include"  \
CONFIGURE_OPTS="--with-openssl=/packages/run.64/openssl-1.1.1"  \
LD_RUN_PATH=$LIBRARY_PATH \
pyenv install -v 3.7.3
pyenv global 3.7.3
pyenv local 3.7.3
```

Now you can install additional python packages, including jupyter and others that you may find helpful:

```bash 
pip install jupyter pandas matplotlib numpy networkx graphviz scipy coloredlogs mysqlclient \
    requests sarge cryptography paramiko shellescape
```

## Running Jupyter notebook

In order to run Jupyter notebook on the server you need to start it in a `screen` or `tmux` so the process runs in the background and survives SSH session termination.

Then you need to specify a port which will Jupyter use for incoming connections. If there are several users sharing the server for the same purpose of Jupyter notebook sharing you should use unique port number.

Create following file `jupyter.sh` on the server:

```bash 
#!/bin/bash
# jupyter notebook password
# jupyter notebook list
: "${JPORT:=8876}"  # unique port per user

# Dependency loading for MUNI RHEL8 machines
module load openssl-1.1.1 zlib-1.2.11 ncurses-5.9 readline-7.0 libffi-3.2.1 bzip2 xz-5.2.4 sqlite-3.29.0 mariadb-client-8.0.17 graphviz-2.26.3 

# Run the notebook in the terminal
nice -n 20 jupyter-notebook --no-browser --port $JPORT .
```

Then run the script `jupyter.sh` in the `screen`:

```bash 
screen -mS jupyter  # creates a new screen named jupyter, attaches the screen
bash ./jupyter.sh   # this is executed in the screen

# Copy the login link returned by the jupyter process - you will need it to login to the notebook later.
# Press CTRL + A + D to detach the screen, keeps jupyter running in the background.
```

Now the jupyter is running on the server loopback interface, namely `127.0.0.1:$JPORT`.

Screen runs until it is terminated or until server restarts. If you want to connect to existing screen session use:

```bash 
screen -x jupyter
```

### Aura SSH tunnel (only for MUNI setup)

As the Aura server is accessible from the MUNI network only, it may be useful to setup automated forwarding via Aisa server as it is accessible from all IPs.

In order to setup the tunnel, edit `~/.ssh/config`:

```
host aura
hostname aura.fi.muni.cz
user YOUR_LOGIN_NAME
ProxyCommand ssh -q -W %h:%p YOUR_LOGIN_NAME@aisa.fi.muni.cz
``` 

### Connecting to the Jupyter notebook

In order to connect to the notebook via browser you need to establish a SSH tunnel to the server.
Run the following command locally:

```bash 
: "${JPORT:=8876}"
ssh -L $JPORT:localhost:$JPORT aura -t 'jupyter notebook list; bash -l'
```

This command opens a SSH connection and creates a tunnel to the jupyter notebook socket running on the server. 
Now you can visit `http://127.0.0.1:$JPORT` locally, in your web browser. However, to login to the notebook you need to use login link provided by the jupyter notebook after start on the server.

If you don't have local SSH client you can use Putty for example, but don't forget to setup local forwarding `$JPORT:localhost:$JPORT`. Please note that the `$JPORT` is not defined on your workstation so either define it correctly or substitute it with the corresponding port number.

### Jupyter notebook files

Keep in mind that once you connect to the remotely running jupyter notebook your notebook files are stored on the server.
You may need additional mechanisms to keep these notebooks versioned, backed up or synced with your local files (out of the scope of this tutorial).

One workaround to keep local notebooks synced with the remote ones is to use `SSHFS` to map remote file system to your folder.

## Practical usage

Now you can run experiments in Jupyter notebooks remotely and uninterrupted on a powerful server (400 GB of RAM).
My general practice is to extract stable pieces of code to separate python packages and import the packages in the notebook so it does not grow large beyond the point of maintainability. 

The packages should be either installed via pip or placed to the same folder where the notebook / jupyter runs from (i.e., where you start `jupyter.sh` script). 

If you change the python code outside the notebook (e.g., package update) you need to restart the Jupyter kernel (e.g., via web interface) so Jupyter loads new code version.

### Troubleshooting

*SSL module could not be compiled*

Try: `echo $LIBRARY_PATH` to make sure you have `openssl-1.1.1` as the first element on this path. 
Unload all other openssl versions present.

### Summary

In order to run the Jupyter on the server we needed to perform the following one-time-only tasks:
- Install [pyenv]
- Build and install local Python 3.7.1 with [pyenv]
- Install python packages we need to use

The task needed once per server restart:
- Start the screen with the jupyter notebook process on the server

Each time your workstation drops network connection:
- Connect to the server with the SSH which creates the local TCP tunnel so you can access the Jupyter notebook web server on your workstation.


### Notes for building dependencies from the source

- Build deps with `-fPIC`, position independent code, so the linking to shared libs works


Example how to build readline-8.0 on RHEL:

- Compile `ncurses-5.9` with `-fPIC`
- Compile readline:

```bash
CFLAGS="-fPIC -I/packages/share/ncurses-5.9/include" \
LDFLAGS="-L/packages/run.64/ncurses-5.9/lib"  \
CPPFLAGS="-fPIC -I/packages/share/ncurses-5.9/include" \
SHLIB_LIBS="-lncurses" \
./configure --prefix=/packages/share/readline-8.0/ --exec-prefix=/packages/run.64/readline-8.0/ --with-curses
make -j44 VERBOSE=1

# re-link final libraries manually, add -lncurses
cd shlib
gcc -shared -Wl,-soname,libreadline.so.8.0 -L/packages/run.64/ncurses-5.9/lib -Wl,-rpath,/packages/run.64/readline-8.0/lib -Wl,-soname,`basename libreadline.so.8.0 .0` -o libreadline.so.8.0 readline.so vi_mode.so funmap.so keymaps.so parens.so search.so rltty.so complete.so bind.so isearch.so display.so signals.so util.so kill.so undo.so macro.so input.so callback.so terminal.so text.so nls.so misc.so history.so histexpand.so histfile.so histsearch.so shell.so mbutil.so tilde.so colors.so parse-colors.so xmalloc.so xfree.so compat.so -lncurses
gcc -shared -Wl,-soname,libhistory.so.8.0 -L/packages/run.64/ncurses-5.9/lib -Wl,-rpath,/packages/run.64/readline-8.0/lib -Wl,-soname,`basename libhistory.so.8.0 .0` -o libhistory.so.8.0 history.so histexpand.so histfile.so histsearch.so shell.so mbutil.so xmalloc.so xfree.so -lncurses

make install
```

Module file example:

```
#%Module1.0
#!
#! Title: readline
#! Platforms: rhel8
#! Version: 8.0
#! Description: Console lib
#!
#!
#!
proc ModulesHelp {} {
global ModulesCurrentModulefile
puts stdout "modulehelp $ModulesCurrentModulefile"
}

module add pkg-config

prepend-path PATH               /packages/run.64/readline-8.0/bin
prepend-path LD_LIBRARY_PATH    /packages/run.64/readline-8.0/lib
prepend-path MANPATH            /packages/share/readline-8.0/share/man
prepend-path C_INCLUDE_PATH     /packages/share/readline-8.0/include
prepend-path CPLUS_INCLUDE_PATH /packages/share/readline-8.0/include
prepend-path LIBRARY_PATH       /packages/run.64/readline-8.0/lib
prepend-path PKG_CONFIG_PATH    /packages/run.64/readline-8.0/lib/pkgconfig
```


[pyenv]: https://github.com/pyenv/pyenv
