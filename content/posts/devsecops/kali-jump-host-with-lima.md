---
title: "Running a Kali Linux Jump Host with Lima"
description: "Set up a headless Kali Linux VM on an Apple Silicon Mac with Lima, run security tools from your terminal, and connect the VM to Claude Code through MCP."
date: 2026-10-08
lastmod: 2026-10-08
draft: true
sidebar: "right"
widgets:
  - "ddg-search"
  - "recent"
  - "social"
toc: true
tags:
  - "lima"
  - "kali linux"
  - "qemu"
  - "claude code"
  - "mcp"
---

While I was working on [Evaluating GLM-5.3 Cybersecurity Capabilities](https://deployment.properties/posts/devsecops/glm-5-3-cybersecurity-capabilities/), I learned a helpful trick to run a local [Kali Linux](https://www.kali.org/) jump host with [Lima](https://lima-vm.io/). I mentioned the setup in a gist linked in that previous post, but I thought it would be cool to elaborate on the details in a separate post. If you are interested in this subject, you probably understand the motivation. It's a nice and quick way (once you have the template created) to get an ephemeral Kali box ready to use when you need security tools.

<!--more-->

[Lima (Linux Machines)](https://lima-vm.io/) is an open-source command-line tool that launches lightweight Linux virtual machines and manages their configuration, startup, shell access, file sharing, and port forwarding. You still have a VM consuming CPU, memory, and disk; the appeal here is a headless environment, without heavier virtualization software.

Use Kali as a local jump host: a Linux machine from which you connect to lab targets and run security tools. The configuration below targets an Apple Silicon Mac with an ARM64 guest.

## Install Lima

Follow the [official Lima installation documentation](https://lima-vm.io/docs/installation/) for your platform and package manager. On macOS with Homebrew, install Lima and QEMU, the virtualization backend used by the Kali template:

```bash
# on macOS
brew install lima qemu
```

QEMU also supplies `qemu-img`, which converts the downloaded Kali disk into the format used below. Lima supports [other VM backends](https://lima-vm.io/docs/config/vmtype/), including Apple's Virtualization.framework; this template explicitly selects QEMU to follow the original setup.

Verify that both command-line tools are available:

```bash
# on macOS
limactl --version
qemu-img --version
```

Both commands should print a version. Use Lima 2.0 or later for the template and the MCP integration at the end of the post.

Before building the Kali template, create a demo VM from Lima's default template and run a command inside it:

```bash
# on macOS
limactl create --name demo template:default
limactl start demo
limactl shell demo uname -a
```

The final command should report a Linux kernel. `limactl create` defines the instance, `limactl start` boots it, and `limactl shell` executes a command inside the guest.

Open an interactive shell in the demo VM:

```bash
# on macOS
limactl shell demo
```

The command gives you a prompt inside the VM. Run `exit` to return to macOS.

## Build a Kali template

A Lima template is a YAML file describing the guest image, resources, and provisioning commands. For Kali, start with its generic cloud image, which supports automated initialization through `cloud-init`.

### Prepare the disk image

Create a working directory and download the Kali 2026.2 ARM64 archive used in the original gist:

```bash
# on macOS
mkdir -p ~/lima-kali
cd ~/lima-kali
curl -fLO \
  https://kali.download/cloud-images/kali-2026.2/kali-linux-2026.2-cloud-genericcloud-arm64.tar.xz
```

The archive contains the guest's disk image. The versioned URL fixes the base image for this example; Kali's rolling package repository still determines which package versions provisioning installs.

Verify the archive against the ARM64 entry in Kali's [published SHA-256 checksums](https://kali.download/cloud-images/kali-2026.2/SHA256SUMS):

```bash
# on macOS, in ~/lima-kali
shasum -a 256 -c <<'EOF'
9ab19c28d049fdc6f1d5d30f6cc93b8b01997f11c89f6992f690c07b16b7b7e4  kali-linux-2026.2-cloud-genericcloud-arm64.tar.xz
EOF
```

The result should end in `OK`. If it reports a mismatch, resolve that before extracting the image.

Extract the raw disk, convert it to QCOW2, QEMU's disk-image format, and inspect the result:

```bash
# on macOS, in ~/lima-kali
tar -xJf kali-linux-2026.2-cloud-genericcloud-arm64.tar.xz
qemu-img convert -p -f raw -O qcow2 disk.raw kali.qcow2
qemu-img info kali.qcow2
```

The last command should report `file format: qcow2`. Keep `kali.qcow2` beside the template so its relative image path resolves correctly.

### Define the VM

Save the following as `~/lima-kali/kali.yaml`. It follows the gist's configuration and explicitly includes `tcpdump` for the packet-capture example below:

```yaml
minimumLimaVersion: "2.0.0"

vmType: "qemu"
arch: "aarch64"

images:
  - location: "./kali.qcow2"
    arch: "aarch64"

cpus: 4
memory: "6GiB"
disk: "100GiB"

# Keep the Mac home directory outside the guest.
mounts: []

# Do not forward the host SSH agent or import its public keys.
ssh:
  forwardAgent: false
  loadDotSSHPubKeys: false

containerd:
  system: false
  user: false

provision:
  - mode: system
    script: |
      #!/bin/bash
      set -euxo pipefail
      export DEBIAN_FRONTEND=noninteractive

      apt-get update
      apt-get install -y \
        kali-linux-headless \
        docker.io \
        tcpdump \
        jq \
        git \
        curl \
        wget \
        tmux \
        ripgrep \
        python3 \
        python3-pip \
        python3-venv \
        build-essential

      systemctl enable --now docker
      usermod -aG docker "{{.User}}"

probes:
  - mode: readiness
    description: "Kali security tools installed"
    script: |
      #!/bin/bash
      set -eu
      command -v nmap >/dev/null
      command -v curl >/dev/null
      command -v tcpdump >/dev/null
      command -v python3 >/dev/null

message: |
  Kali Lima VM is ready.
  Enter with: limactl shell {{.Name}}
```

The `kali-linux-headless` metapackage installs Kali's command-line tool collection without a desktop. The `provision` script installs packages inside the guest, and the readiness probe checks that the tools used in this post are available.

The resource values are a starting allocation: four virtual CPUs, 6 GiB of memory, and a 100 GiB virtual disk. Adjust them for your laptop and workload; running several scans or fuzzers concurrently can exhaust the guest's resources.

The template disables [host-directory mounts](https://lima-vm.io/docs/config/mount/) and SSH agent forwarding. Lima still creates its own SSH access for managing the guest. Docker follows the original gist and is optional: remove `docker.io`, `systemctl enable --now docker`, and `usermod` if you do not need containers inside Kali.

### Start and verify Kali

Validate the template, then create and start an instance named `kali`:

```bash
# on macOS
cd ~/lima-kali
limactl validate ./kali.yaml
limactl start --name kali ./kali.yaml
```

The first boot downloads and installs the tool collection, so allow time for provisioning. Wait for Lima's readiness message before running commands; if validation or provisioning fails, resolve the reported error before continuing.

Confirm the operating system and installed tools from the host:

```bash
# on macOS; each command executes inside Kali
limactl shell kali cat /etc/os-release
limactl shell kali nmap --version
limactl shell kali python3 --version
```

The first command should identify Kali Linux, and the others should print their installed versions. To work interactively, run `limactl shell kali`.

## Use the Kali template

Run tools through `limactl shell kali` from macOS, or enter the guest and use them directly. The prefix selects where the command executes; the arguments after it belong to the guest command.

### Scan Nmap's test host

The [Nmap examples documentation](https://nmap.org/book/man-examples.html) provides `scanme.nmap.org` for testing. Its permission covers Nmap scans, excludes exploitation and denial-of-service testing, and limits use to a dozen scans per day.

Run a TCP connect scan against three ports, with service-version detection:

```bash
# on macOS; Nmap executes inside Kali
limactl shell kali nmap -sT -Pn -sV -p 22,80,443 scanme.nmap.org
```

The `-sT` flag uses ordinary TCP connections without requiring root, `-Pn` skips host discovery, and `-sV` probes open ports for service information. Read the reported port states and service details; results depend on what the target exposes when you run the scan.

### Inspect an HTTP response

Use [`curl`](https://curl.se/docs/manpage.html) to request response headers from an example website:

```bash
# on macOS; curl executes inside Kali
limactl shell kali curl -I https://example.com
```

The `-I` flag sends an HTTP `HEAD` request. The response shows the status code and headers, providing a check that the guest can resolve a public hostname and establish an HTTPS connection.

### Observe the guest's network traffic

Use [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) to capture HTTP traffic inside Kali. In one terminal, start a capture that stops after ten matching packets:

```bash
# on macOS, terminal 1; capture runs inside Kali
limactl shell kali sudo tcpdump -i any -nn -c 10 'tcp port 80'
```

The `-i any` flag selects the guest's network interfaces, `-nn` keeps addresses and ports numeric, and `-c 10` limits the capture. The command waits until traffic matches its filter.

In a second terminal, generate an HTTP request:

```bash
# on macOS, terminal 2; request runs inside Kali
limactl shell kali curl -I http://example.com
```

The first terminal should show the connection's packets. If fewer than ten packets arrive, stop the capture with `Ctrl+C`; this observes the VM's traffic, not every connection made by the laptop.

### Reach a lab running on macOS

Inside Kali, `localhost` and `127.0.0.1` refer to Kali. With Lima's [default user-mode network](https://lima-vm.io/docs/config/network/user/), `host.lima.internal` provides access to the host's loopback services.

If your lab application is already exposed on macOS port `3000`, inspect it from Kali:

```bash
# on macOS; request originates inside Kali
limactl shell kali curl -I http://host.lima.internal:3000/
```

An HTTP response confirms the VM can reach the application. A connection error points to the service, its published port, or the network path; changing the URL to `localhost` would instead ask Kali for that service.

This setup suits command-line network and web testing, but it does not place Kali directly on your physical LAN. For targets that must initiate connections back to Kali, plan the listener's reachable address and port; consult Lima's [networking](https://lima-vm.io/docs/config/network/) and [port-forwarding documentation](https://lima-vm.io/docs/config/port/) for the required configuration.

## Bonus: connect Kali to Claude Code through MCP

Lima's [Model Context Protocol integration](https://lima-vm.io/docs/config/ai/outside/) exposes tools for executing commands and reading or writing files inside a VM. [Claude Code](https://code.claude.com/docs/en/mcp) can use those tools while continuing to run on macOS.

Confirm that Lima's MCP plugin is installed:

```bash
# on macOS
limactl mcp -v
```

The command should print the plugin version. The plugin is bundled with Lima 2.0 and later, although some installation methods omit it; if the command is unavailable, check Lima's [MCP prerequisites](https://lima-vm.io/docs/config/ai/outside/gemini/#prerequisite) and installation instructions.

With Kali running, register its MCP server from the project where you want Claude Code to use it:

```bash
# on macOS, from your Claude Code project directory
claude mcp add --transport stdio --scope project kali -- \
  limactl mcp serve kali
claude mcp list
```

The `kali` server should appear in the list. This registers Lima's existing MCP server with Claude Code; it does not install another service inside Kali. The `--scope project` flag stores the configuration in the project's `.mcp.json` file, and `--transport stdio` connects through the server process's standard input and output.

Start Claude Code from the same project directory:

```bash
# on macOS, from your Claude Code project directory
claude
```

Approve the project MCP server if prompted, then run `/mcp` inside Claude Code to check its connection. Ask Claude to use the `kali` MCP server to inspect `/etc/os-release` and run `nmap --version`; those operations execute inside the VM.

Claude Code can also use its host shell to run the same `limactl shell kali ...` commands shown throughout this post. MCP provides a tool interface for the VM, while the shell approach remains available. Claude's own shell commands still execute on macOS unless they explicitly enter Kali, so make the intended execution environment part of your project instructions.

## Wrapping up

The setup provides a reusable environment for working with security tools:

- **Lima**: manages the Linux VM and provides shell and SSH access from your terminal.
- **Kali template**: defines the base image, resource allocation, and guest provisioning without a desktop environment.
- **Guest networking**: separates Kali's `localhost` from the laptop and provides `host.lima.internal` for local lab access.
- **Claude Code integration**: makes the same environment available through MCP or host-shell commands that call `limactl`.

Stopping the VM preserves its installed tools and guest files; restart it with `limactl start kali`. Deleting the instance removes that guest state, so copy out any results you want to keep first. Treat this as a local study environment; keep targets separate when your experiment depends on testing them only over the network.

When you are finished with the environment, remove the project MCP entry and delete the tutorial's VMs:

```bash
# on macOS, from the project directory used for MCP
claude mcp remove --scope project kali
limactl stop demo
limactl delete demo
limactl stop kali
limactl delete kali
```

These commands remove the MCP registration and Lima instances. The original `kali.yaml` and `kali.qcow2` files in `~/lima-kali` remain available to create another instance.

> **AI assistance acknowledgment**: This article was produced with AI assistance. AI helped draft the tutorial, check commands against the referenced documentation, and refine the wording. The thesis, argument, editorial decisions, and final responsibility are mine.
