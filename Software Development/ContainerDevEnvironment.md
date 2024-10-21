# VsCode Flatpak + Podman on an Immutable System

**Immutable** means that the root folder is in read-only mode and cannot be written to by any user, potentially not even by the root user (although the root user can disable immutability).

In this scenario, setting up a development environment is a bit more complex. Here are some notes on the setup process.

## Starting point

- development in containers
- Podman as a containers manager
- Flatpak VsCode + dev-containers extension (all other extensions must be container-specific[^1])

[^1]: Installed inside the container and accessible only from within it.

> Note: most of the material comes from [Zihad guide](https://www.zihad.com.bd/posts/setup-vscode-flatpak-devcontainer-with-podman/) published in August 2024. So, it's pretty recent.

## The guide

### Install flatpak vscode and podman

```shell
flatpak install --user -y \
com.visualstudio.code \
com.visualstudio.code.tool.podman
```

The podman tool is essential in order to allow to flatpak vscode to communicate with a podman socket on the host and must be installed _within the flatpaks system_. In fedora Silverblue 'podman' comes pre-installed. **This guide assumes that podman is already installed on the host system**. By giving the command 'com.visualstudio.code.tool.podman', you are probably asked to indicate one of the available versions. In my experience, the stable version does not work and one must opt for the 24.

### Start podman socket

```shell
systemctl --user enable --now podman.socket && \
systemctl --user status podman.socket
```

The first command start podman.socket every time a session in accessed, and the flag '--now' make sure that the socket start right after the command. The second command shows the status of podman.socket.

### Setup some vscode flatpak settings

Being containerized as a flatpak, we need to setup some permission on some host directory to allow vscode to interact with podman.socket.

```shell
flatpak override --user \
--filesystem=xdg-run/podman:ro \
--filesystem=/run/user/$UID/podman/podman.sock:ro \
--filesystem=/tmp:rw \
com.visualstudio.code
```

### Test podman socket

```shell
curl -w "\n" \
-H "Content-Type: application/json" \
--unix-socket /run/user/$UID/podman/podman.sock \
http://localhost/_ping
```

This command uses `curl` to send a HTTP request to podman unix socket to verify if it's running correctly. If everything is fine, we are going to get an **OK** as an answer.


Now most of the heavy lifting is done. We just need to install `dev-containers` vscode extension and force di editor to use podman instead of the default `docker.socket`.

### Install `dev-containers` extension

```shell
flatpak run --user --command=sh com.visualstudio.code -c \
"/app/extra/vscode/bin/code --install-extension ms-vscode-remote.remote-containers"
```

This command may seem a bit complicated because we are using the Flatpak version of VSCode. However, it is equivalent to manually installing the extension directly from the VSCode interface.

### Configure VsCode to use podman instead of docker

From the terminal

```shell
mkdir -p ~/.var/app/com.visualstudio.code/config/Code/User
nano ~/.var/app/com.visualstudio.code/config/Code/User/settings.json
```

and adding the following setting to the .json file

```json
{
  "dev.containers.dockerPath": "/app/tools/podman/bin/podman-remote"
}
```

This is absolutely equivalent to 

- open vscode
- press `control + p` (windows,linux) or `cmd-p` (macos) to oper User Settings (JSON)
- paste the previous setting and save.

### Test vscode-podman connection

```shell
flatpak run --user --command=sh com.visualstudio.code
/app/tools/podman/bin/podman-remote version
/app/tools/podman/bin/podman-remote run --rm hello-world
```

The command download hello-world image and crete and run a container from that.

## Other Resources

- [Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)
- [Dev Containers tutorial](https://code.visualstudio.com/docs/devcontainers/tutorial)
- [Podman Tutorials](https://docs.podman.io/en/latest/Tutorials.html)
- [Fedora Silverblue](https://docs.podman.io/en/latest/Tutorials.html)
- [What is an immutable distro](https://itsfoss.com/immutable-distro/)
