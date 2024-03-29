---
date: 2024-02-07
---

# PandaRoot container for WSL

This page describes how to build and and run [PandaRoot](https://git.panda.gsi.de/PandaRootGroup/PandaRoot) inside a Docker container on [Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/about) (WSL). Install WSL following [these official instructions](https://learn.microsoft.com/en-us/windows/wsl/install). It's best to stick to the defaults: just chose Ubuntu as Linux distro. If you installed [Visual Studio Code](https://code.visualstudio.com), have a look at [this tutorial](https://code.visualstudio.com/docs/remote/wsl) on WSL and VSCode.

## 1. Docker container

The first part of these instructions focus on terminal commands only. The goal is to build the [PandaRoot](https://git.panda.gsi.de/PandaRootGroup/PandaRoot) source code within a Docker container that has [FairROOT](https://fairroot.gsi.de) installed. In [](#2-vscode-dev-container), we automate this process, so that the container launches directly when you start [VSCode](https://code.visualstudio.com).

### 1.1. Install Docker

In a terminal in WSL, install [Podman](https://podman.io), which is can be used to run Docker containers without admin (`sudo`) rights.

```bash
sudo apt install podman-docker
```

::::{tip} Tips about Docker
:class: dropdown
**Images** are immutable files that are downloaded from [Docker Hub](https://hub.docker.com) or other registries. Once a specific tag of an image is downloaded, it is stored and you don't need to download it again. The downloaded images can be listed with

```bash
docker images -a
```

**Containers** are created once you _run_ an image. An example is the following command, which runs an image for the latest release of Debian:

```bash
docker run --rm -it docker.io/debian:latest bash
```

:::{important}
Images and containers can take up a lot of space, so you want to remove them as soon as you don't run them anymore. The `--rm` removes the container after you close it. See [this nice explainer](https://youtu.be/0vxIyXgkihA) on YouTube for more info.
:::
::::

### 1.2. FairSoft container with PandaRoot

If you haven't already, clone the [PandaRoot source code](https://git.panda.gsi.de/PandaRootGroup/PandaRoot):

```bash
git clone git@git.panda.gsi.de:PandaRootGroup/PandaRoot
```

and navigate into the cloned repository:

```bash
cd PandaRoot
```

Next, run [this FairSoft Docker image](https://hub.docker.com/r/tstockmanns/fairroot_v18_6_fairsoft_apr22_ubuntu_22) and bind (`-v`) the current working directory with the PandaRoot source code to the container: <!-- cspell:ignore tstockmanns -->

```bash
docker run --rm -it \
  -v "$(pwd)":/mnt/work/PandaRoot \
  -w /mnt/work/PandaRoot \
  tstockmanns/fairroot_v18_6_fairsoft_apr22_ubuntu_22
```

You now have a terminal before you inside the FairSoft container, with access to the cloned PandaRoot source code.

### 1.3. Build and test PandaRoot

You are now ready to build PandaRoot using [these (internal) instructions on GitLab](https://git.panda.gsi.de/PandaRootGroup/PandaRoot/-/blob/4b8df57/docs/Installation/Install_Developers.rst) (see "Out-of-build installation" in particular):

```bash
mkdir build install -p
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=../install
make -j$(nproc) install
```

:::{admonition}
The build process that starts when you run `make` can take up to **10&nbsp;min**... ☕
:::

Once done, navigate back to the PandaRoot main directory and source the installed configuration:

```bash
. /mnt/work/PandaRoot/install/bin/config.sh -p
```

You can test whether the installation works with the [`rho/tut_sim.C`](https://git.panda.gsi.de/PandaRootGroup/PandaRoot/-/blob/4b8df57/tutorials/rho/tut_sim.C) script:

```bash
cd /mnt/work/PandaRoot/tutorials/rho
root -l -b -q tut_sim.C
```

After some minutes of generating events, you'll see something like:

```
....
<DartMeasurement name="MaxMemory" type="numeric/double">915.203</DartMeasurement>
<DartMeasurement name="CpuLoad" type="numeric/double">0.992</DartMeasurement>

Output file is          ./signal_sim.root
Parameter ROOT file is  ./signal_par.root
Parameter ASCII file is all.par
Real time 66.462 s, CPU time 65.940s
CPU usage 99.214%
Max Memory 915.203 MB
Macro finished successfully.
double free or corruption (!prev)
```

That means you're good to go! 🎉

## 2. VSCode Dev Container

Visual Studio Code can be run inside a Docker container, so that you don't have to run the commands described in [](#1-docker-container). As of writing, the [`.devcontainer.json`](https://git.panda.gsi.de/PandaRootGroup/PandaRoot/-/blob/4b8df57/.devcontainer.json) file in PandaRoot does not work on WSL. To fix this, do the following:

1. Install the VSCode [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) extension.

2. Replace the contents of the `.devcontainer.json` file in the PandaRoot repository with the following:

   ```{code} json
   :filename: .devcontainer.json
   {
     "image": "tstockmanns/fairroot_v18_6_fairsoft_apr22_ubuntu_22",
     "postStartCommand": ". /mnt/work/PandaRoot/install/bin/config.sh",
     "workspaceFolder": "/mnt/work/PandaRoot",
     "workspaceMount": "source=${localWorkspaceFolder},target=/mnt/work/PandaRoot,type=bind,consistency=cached"
   }
   ```

3. **Bonus**: To avoid always having to run the `config.sh` script when you launch the terminal, set the following in the [settings for your VSCode workspace](https://code.visualstudio.com/docs/getstarted/settings):

   ```{code} json
   :filename: .vscode/settings.json
   {
     "terminal.integrated.defaultProfile.linux": "PandaRoot",
     "terminal.integrated.profiles.linux": {
       "PandaRoot": {
         "args": ["-l", "-c", ". /mnt/work/PandaRoot/install/bin/config.sh; bash"],
         "path": "/bin/bash"
       }
     }
   }
   ```
