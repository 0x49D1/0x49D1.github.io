---
layout: post
title: "How to Enable Remote Debugging in a Docker Container for Rider and Visual Studio"
date: 2025-02-21
excerpt: "When containerizing your .NET application, you might still need to attach your IDE’s debugger to troubleshoot issues that occur only in the containerized environment. In this article, we show you how to add remote debugging capabilities to your Docker container so that both JetBrains Rider and Visual Studio can connect seamlessly. Keep in mind that such debugging sessions are rare, but sometimes very useful."
author: "Dimitri Pursanov"
tags: [csharp, remote, debugging, dotnet]
---

# How to Enable Remote Debugging in a Docker Container for Rider and Visual Studio

When containerizing your .NET application, you might still need to attach your IDE’s debugger to troubleshoot issues that occur only in the containerized environment. In this article, we show you how to add remote debugging capabilities to your Docker container so that both JetBrains Rider and Visual Studio can connect seamlessly. Keep in mind that such debugging sessions are rare, but sometimes very useful.

We’ll cover how to set up a secure SSH environment in the container, install the remote debugger (vsdbg), and configure your container with best practices for naming images and containers. The same kind of instructions should work in Kubernetes pods.

## Why Remote Debugging in Docker?

Remote debugging lets you diagnose issues that appear only when your application is running inside a container. Using SSH as a secure tunnel has multiple benefits:

- **Security**: SSH encrypts all communications, keeping your debugging session secure.
- **Ease of Deployment**: Your IDE can deploy or attach the remote debugging tools (like vsdbg) via SSH.
- **Network Flexibility**: Even if your container is behind a firewall or on a host with multiple containers, you can expose only a single port (usually SSH on port 22) and tunnel the debug port internally.

## How It Works

- **Build and Deploy with Debug Symbols**: When you compile your .NET application with debug symbols (PDB files), the debugger can correlate your running code with your source code.
- **Remote Debugger Agent (vsdbg)**: The vsdbg tool acts as the remote debugging agent. It runs on the container (for example, listening on port 4022) and communicates with your IDE once attached.
- **SSH for Secure Connectivity**: Instead of exposing the debugger port directly, you install an SSH server inside the container. Your IDE connects via SSH (on port 22) and tunnels the debug connection securely.
- **IDE Integration**: Both JetBrains Rider and Visual Studio can attach to a remote process via SSH. Rider supports SSH-based remote debugging out-of-the-box, while Visual Studio can use an SSH tunnel (manually set up or via its Linux connection features) to connect to vsdbg.

```
       +------------------------------------------------+
       |          Developer Machine (IDE)             |
       |  (JetBrains Rider or Visual Studio Debugger)  |
       +----------------------+-------------------------+
                              │
                              │ SSH Connection (port 22 on host)
                              │
       +----------------------+-------------------------+
       |                Docker Host                   |
       |  (Multiple Containers Running on One Server) |
       +----------------------+-------------------------+
                              │
                              │ Container Networking & Port Mapping
                              │ (e.g., host port 2022 → container port 22)
                              │
       +----------------------+-------------------------+
       |            Docker Container                  |
       |                                              |
       |  +-----------------------------------------+ |
       |  |  OpenSSH Server (sshd)                  | |
       |  |     - Accepts SSH connections           | |
       |  +-----------------------------------------+ |
       |                                              |
       |  +-----------------------------------------+ |
       |  |  Remote Debugger Agent (vsdbg)          | |
       |  |     - Listens on internal port 4022     | |
       |  +-----------------------------------------+ |
       |                                              |
       |  +-----------------------------------------+ |
       |  |  .NET Application with Debug Symbols    | |
       |  |     - Runs on port 8080 (for example)   | |
       |  +-----------------------------------------+ |
       +----------------------------------------------+
```

## Sample Dockerfile for Remote Debugging

Below is a streamlined Dockerfile that sets up a remote debugging environment. We’ve chosen community-friendly names — using `test-remote-debug` for the image and `test-remote-debugger` for the container. Dotnet project was called as sample `TestRemoteDebug`, which is just a sample WebAPI project for us to “debug”.

```dockerfile
# Use the official ASP.NET runtime (8.0) as our base image.
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER root
WORKDIR /app

#### Remote Debugging Setup ####
# Update package lists, install openssh-server, curl, and unzip.
RUN apt-get update && \
    apt-get install -y --no-install-recommends openssh-server curl unzip && \
    rm -rf /var/lib/apt/lists/* && \
    mkdir /var/run/sshd && \
    # Set root password to "password123!"
    echo 'root:password123!' | chpasswd && \
    # Configure SSH: allow root login and enable password authentication.
    sed -i 's/^#*PermitRootLogin .*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/^#*PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    echo "PermitUserEnvironment yes" >> /etc/ssh/sshd_config && \
    # Install vsdbg for remote debugging
    curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l /vsdbg && \
    # Prepare SSH environment for root
    mkdir -p /root/.ssh && chmod 700 /root/.ssh
# Expose the SSH port (22) and your application port (e.g. 8080)
EXPOSE 22
EXPOSE 8080
################################

# --- Build and Publish Stages ---
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src/TestRemoteDebug
COPY ["TestRemoteDebug.csproj", "./"]
RUN dotnet restore "TestRemoteDebug.csproj"
COPY . .
RUN dotnet build "TestRemoteDebug.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "TestRemoteDebug.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
# Copy the published application
COPY --from=publish /app/publish .

# Set ASP.NET Core URLs so the app listens on all interfaces
ENV ASPNETCORE_URLS=http://*:8080

# Copy the custom entrypoint script into the container.
COPY ./entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

# Use our entrypoint script to start SSH, vsdbg, and your application.
ENTRYPOINT ["/app/entrypoint.sh"]
```

Create an `entrypoint.sh` file in the same directory as your Dockerfile:

```bash
#!/bin/bash
echo "Starting SSH server..."
/usr/sbin/sshd

echo "Starting vsdbg on port 4022..."
/vsdbg/vsdbg --server --port 4022 --no-https --engineLogging=/tmp/enginelog.log &

# Optional: Wait a few seconds for services to initialize.
sleep 2

echo "Starting TestRemoteDebug application..."
dotnet TestRemoteDebug.dll
```

## Add Debug Symbols even in Release build

Debug symbols are essential because they provide a mapping between the compiled binary and the original source code. Without these symbols, stepping through your code during a debugging session becomes much harder, as variable names and line numbers are missing. It’s important to ensure that your Docker build process includes these symbols, so that when you attach your IDE’s debugger, you get a rich, informative experience. Secure the access to these files (use least privileges, VPN access, SSH Tunnel…), because the information they contain is also valuable to the attacker who wants to reverse-engineer your service.

In the `*.csproj` file you need to add `DebugSymbols`, for like this, to create PDB file:

```xml
<PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <DockerDefaultTargetOS>Linux</DockerDefaultTargetOS>
    <DebugType>portable</DebugType>
    <DebugSymbols>true</DebugSymbols>
</PropertyGroup>
```

## Building and Running the Container

Run the following command in your project directory (where your Dockerfile is located; also we are using `localhost` with docker on local machine here, but server can be remote):

```bash
docker build -t test-remote-debug:latest . && docker run -d -p 8080:8080 -p 2022:22 --name test-remote-debugger test-remote-debug:latest
```

This command does the following:

- Builds the image with the tag `test-remote-debug`.
- Runs the container in detached mode, mapping:
  - Container port 22 (SSH) to host port 2022.
  - Container port 8080 (your application) to host port 8080.
- Names the container `test-remote-debugger`.

## Connecting to Remote Debug

### From JetBrains Rider

1. **Set Up SSH Connection**:
   - Go to Tools → SSH Configurations in Rider.
   - Create a new configuration:
     - Host: `localhost`
     - Port: `2022`
     - Username: `root`
     - Password: `password123!`

2. **Attach to Process**:
   - Use Rider’s Attach to Remote Process feature (via SSH) to connect to your container.
   - Once connected, Rider will list the running .NET processes (using vsdbg on port 4022) for you to select.

### From Visual Studio

1. **Add SSH Connection in “Connection Manager” options part**.
2. **Attach to remote process, choose SSH there and the previously added connection from the list**.

## Pros of having Remote Debug ready production

Remote debugging in production, while risky if not managed properly, offers several advantages when used under controlled conditions:

- **Real-Time Diagnosis**: You can attach a debugger to a live system and inspect its state in real time. This is especially useful when issues occur that are hard to reproduce in development or staging environments.
- **Faster Troubleshooting**: Instead of replicating production conditions in a test environment, you can diagnose problems directly on the production system. This can significantly reduce downtime during critical incidents.
- **Visibility into Production Behavior**: Remote debugging lets you see exactly how the application behaves under real-world load and configuration. This can reveal subtle bugs or performance issues that only manifest in production.
- **Targeted Investigation**: It allows developers to examine live data, state, and configuration to pinpoint the root cause of an issue, which can be particularly valuable for complex, intermittent problems.
- **Non-Invasive Testing (if managed properly)**: With careful configuration (for example, using SSH tunneling), remote debugging can be performed without fully exposing sensitive internals to the internet, keeping access restricted to authorized personnel.

## Cons of having Remote Debug ready production

Remote debugging in production is generally not recommended without strict controls, because it introduces several potential risks and tradeoffs. Here are some of the main considerations:

1. **Security Risks**:
   - **Exposure of Sensitive Information**: When remote debugging is enabled, the debugging agent (e.g. vsdbg) can provide access to detailed application state and code, which could be exploited if an unauthorized user gains access.
   - **Increased Attack Surface**: Even when using SSH tunneling without VPN, misconfigurations (such as weak passwords, untrusted SSH keys, or improperly exposed ports) could allow an attacker to intercept or abuse the remote debug interface.

2. **Performance Impact**:
   - **Potential for Pausing Processes**: Attaching a debugger can halt threads or processes, which in production can cause service disruption or performance degradation.
   - **Resource Overhead**: The remote debugger might consume additional system resources, which could impact overall performance in a production environment.

3. **Operational Complexity**:
   - **Maintenance and Monitoring**: Maintaining a secure remote debugging setup in production requires constant monitoring, ensuring that SSH and debugger agents are up-to-date and correctly configured.
   - **Incident Response**: In the event of a security incident or application crash, having remote debugging enabled may complicate your incident response if an attacker has exploited it.

## Summary

This article showed you how to set up remote debugging in a Docker container by:

- Installing and configuring OpenSSH and vsdbg.
- Using a multi-stage `Dockerfile` to build, publish, and prepare your .NET application.
- Employing best practices for naming your image (`test-remote-debug`) and container (`test-remote-debugger`).
- Explaining how both Rider and Visual Studio can connect to your container via SSH, using port mapping to securely tunnel the remote debugger.

This setup ensures a secure, maintainable, and community-aligned remote debugging environment for your .NET applications running in Docker containers.

## References

1. [JetBrains Rider — SSH Remote Debugging](https://www.jetbrains.com/help/rider/SSH_Remote_Debugging.html)
   This official Rider documentation explains how to set up SSH-based remote debugging, including how vsdbg is used behind the scenes.

2. [Microsoft Documentation — Remote Debugging .NET Core Applications](https://learn.microsoft.com/en-us/visualstudio/debugger/remote-debugging-aspnet-on-a-remote-iis-computer?view=vs-2022)
   Microsoft’s documentation provides an overview of remote debugging for .NET Core applications, including guidance on setting up vsdbg in various environments (Linux, containers, etc.). Although the exact article may be updated over time, you can refer to their Remote Debugging documentation and search for .NET Core or Docker-specific sections.

3. [Visual Studio Code — Remote Development with SSH](https://code.visualstudio.com/docs/remote/ssh)
   For developers who use Visual Studio Code, the Remote Development docs cover how to connect to remote Linux hosts via SSH, which is applicable for remote debugging scenarios.