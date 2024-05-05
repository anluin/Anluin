# Exploring RunC

## What is RunC?

**RunC** is a cornerstone technology in modern containerization, enabling direct execution of containers according to
the Open Container Initiative (OCI) standards. As a stand-alone binary, **RunC** can control the lifecycle of a
container, handling its creation, execution, and termination without the overhead of additional software. This makes it
a key tool for those looking to leverage the core functionalities of containers in customized and optimized
environments.

### Origins and Evolution

Originally a component of Docker, **RunC** was spun out into its own tool as part of the OCI project. This move was
aimed at standardizing container runtime technology across various platforms, thus fostering an ecosystem where
containers are portable and uniform across different systems and infrastructures.

### Why Use RunC?

Here’s why using **RunC** might be beneficial:

- **Lightweight**: It operates directly on the container, without needing a full-fledged daemon.
- **Portability**: Containers managed by RunC can easily be moved between different environments that adhere to OCI
  standards.
- **Flexibility**: Advanced users can manipulate container operations at a granular level.

## Getting Started with RunC

Before diving into container management with RunC, you'll need a container image. This walkthrough uses **skopeo** and
**umoci** to handle image tasks, ensuring that you can pull and unpack container images from a registry.

### Workspace Setup

Creating a dedicated workspace helps in managing containers efficiently:

```sh
mkdir -p workspace/images workspace/bundles
```

### Image Management

Managing container images is a fundamental skill in containerization. In this guide, we'll use **skopeo** and **umoci**,
two powerful tools that provide a streamlined approach to handling container images.

#### Why Skopeo and Umoci?

- **Skopeo**: This command-line utility performs various operations on container images and image repositories. It
  doesn't require the Docker daemon to be running or even installed, which makes it a lighter alternative for just
  handling images. It supports operations like inspecting, copying, deleting images along with transferring images
  between different types of image storages.

- **Umoci**: It is an open-source tool that modifies Open Container Initiative (OCI) images, which are a standardized
  form of container images. This is crucial for altering images in ways that tools like Docker build, traditionally
  handle.

#### Pulling the Image Using Skopeo

```sh
skopeo copy "docker://alpine:latest" "oci:./workspace/images/alpine:latest"
```

**Explanation**:

- **Source**: `"docker://alpine:latest"` tells Skopeo to pull the latest version of the Alpine image from Docker Hub.
- **Destination**: `"oci:./workspace/images/alpine:latest"` instructs Skopeo to save the image in OCI format in the
  specified directory. This path is locally prepared to store the OCI images, exemplifying how Skopeo interacts with
  different storage types (in this case, the OCI layout).

**Benefits**:

- The command abstractly handles authentication and layers, pulling only necessary components.
- Being storage-agnostic, it enables advanced use cases like direct transfers between different storage types, reducing
  the overhead.

#### Unpacking the Image with Umoci

```sh
sudo umoci unpack --image "./workspace/images/alpine:latest" "./workspace/bundles/alpine:latest"
```

**Explanation**:

- `--image "./workspace/images/alpine:latest"` specifies the location and tag of the OCI image.
- `./workspace/bundles/alpine:latest` is the destination directory where the image root filesystem will be extracted to.

**What happens?**:

- **Umoci** unpacks the OCI image format into a runnable bundle that **RunC** can execute. This unpacking includes
  setting up the filesystem hierarchy and extracting image layers.
- It ensures that the unpacked format adheres to the OCI runtime specification, crucial for compatibility across
  different container runtimes.

**Practical insights**:

- This process is vital for transforming a static image into a dynamic, executable environment that RunC can manage.
- Provides the flexibility to modify the image (e.g., configuration files, filesystem structure) before running it,
  which is particularly useful in development and testing.

### Running a Container with RunC

One of the core functionalities of RunC is running containers directly from a command line without any high-level
abstractions or orchestrations.

#### The Command Breakdown

```sh
sudo runc run -b "./workspace/bundles/alpine:latest" "hello_alpine"
```

**Explanation:**

- `sudo`: RunC often requires root privileges to manage various system resources like cgroups and namespaces, ensuring
  that the container has necessary isolation and resource restrictions.
- `runc run`: This is the command to create and run a container.
- `-b "./workspace/bundles/alpine:latest"`: The `-b` option specifies the bundle directory. This directory contains
  the `config.json` file and root filesystem (`rootfs`) which conform to the OCI runtime specification.
  The `config.json` outlines how the container should operate, including process arguments, environment variables, and
  resource limitations.
- `"hello_alpine"`: This is the identifier for the container instance. It should be unique per container instance on the
  host.

#### What Happens When You Run This Command?

1. **Initialization**: RunC reads the `config.json` to understand the particular settings and environment in which the
   container should run. This includes parsing parameters such as namespaces, cgroups settings, security features like
   SELinux labels, and seccomp profiles.

2. **Filesystem Setup**: Using the `rootfs` specified in the bundle, a filesystem is set up and made available to the
   container. This is the container's operating environment.

3. **Process Creation**: RunC then starts the primary process specified in `config.json`. In the case of the Alpine
   image, this is typically the `/bin/sh` shell unless customized otherwise.

4. **Isolation and Resource Allocation**: Establishes namespaces and cgroups as per the configuration to isolate the
   container and limit its resource usage (CPU, memory, IO, etc.).

5. **Execution**: Runs the container's main process. Once this process terminates, the container itself effectively
   ends.

#### Practical Insights

- **Direct Interaction**: This method of running a container is very hands-on, allowing you to specify exactly how the
  container should behave at runtime. It's an excellent opportunity for learning and experimentation with container
  fundamentals.

- **Quick Testing**: Running a minimal container like Alpine is fast and useful for quick tests or for developing
  applications that require a controlled environment.

- **Scripting and Automation**: Running containers with RunC can be scripted in bash or another shell, allowing for
  automated setup and teardown of containers in testing pipelines or development environments without requiring complex
  orchestration tools.

- **Security Context**: Running containers with `sudo` highlights the importance of considering security. It's essential
  to manage permissions carefully, possibly setting up user namespaces to limit potential damage a container can do to
  the host system.

## Networking: Container and Host Communication

Initially, our container is isolated. We'll configure networking to allow communication between the container and the
external world, enhancing its capabilities.

### Network Setup

We create a bridge and connect it to a virtual ethernet (veth) pair, bridging the gap between the container and the host
network.

#### Setup

```sh
sudo brctl addbr runc0 && \
sudo ip link set runc0 up && \
sudo ip addr add 192.168.10.1/24 dev runc0 && \
sudo ip link add name veth-host type veth peer name veth-guest && \
sudo ip link set veth-host up && \
sudo brctl addif runc0 veth-host && \
sudo ip netns add runc && \
sudo ip link set veth-guest netns runc && \
sudo ip netns exec runc ip link set veth-guest name eth1 && \
sudo ip netns exec runc ip addr add 192.168.10.101/24 dev eth1 && \
sudo ip netns exec runc ip link set eth1 up && \
sudo ip netns exec runc ip route add default via 192.168.10.1
```

###### Cleanup

```sh
sudo ip link set runc0 down && \
sudo brctl delbr runc0 && \
sudo ip netns del runc && \
sudo ip link del veth-host
```

### Bundle Configuration Adjustments for Network Access

When running containers with **RunC**, adjusting the configuration to specify how the container interacts with various
system resources, including the network, is crucial. Below we discuss how to modify the `config.json`, which controls
various settings like names and namespaces.

#### Understanding the Script

The command provided uses a combination of shell scripting, **Deno** (a secure runtime for JavaScript and TypeScript),
and JSON manipulation to programmatically update the `config.json` file stored within a container's bundle. Here's what
each part does:

```sh
cat <<END | sed 's/^[ \t]*//' | sudo deno run -A /dev/stdin > /dev/null
    // Load and parse the existing config.json file
    const config = await Deno.readTextFile("./workspace/bundles/alpine:latest/config.json").then(JSON.parse);
    
    // Locate the network namespace configuration section
    const networkNamespaceConfig = config.linux.namespaces.find(namespace => namespace.type === "network");
    
    // Enhance the container's capabilities to include network operations 
    config.process.capabilities.bounding = [...new Set([...config.process.capabilities.bounding, "CAP_NET_RAW"])];
    config.process.capabilities.effective = [...new Set([...config.process.capabilities.bounding, "CAP_NET_RAW"])];
    config.process.capabilities.inheritable = [...new Set([...config.process.capabilities.bounding, "CAP_NET_RAW"])];
    config.process.capabilities.permitted = [...new Set([...config.process.capabilities.bounding, "CAP_NET_RAW"])];
    config.process.capabilities.ambient = [...new Set([...config.process.capabilities.bounding, "CAP_NET_RAW"])];
    
    // Adjust the network namespace
    networkNamespaceConfig.path = "/var/run/netns/runc";
    
    // Write the modified configuration back to the file
    await Deno.writeTextFile("./workspace/bundles/alpine:latest/config.json", JSON.stringify(config, null, 4));
END
  ```

#### Breakdown of Changes

**Capabilities**:

- The script enhances the container's Linux capabilities to include `CAP_NET_RAW`, which allows it to use networking
  features like ICMP (used for ping) and raw socket operations. This change is crucial for network troubleshooting and
  interfacing directly with network protocols.

**Network Namespace**:

- By modifying the `networkNamespaceConfig.path`, the script sets the container to utilize the newly created network
  namespace `runc`. This namespace is where we've set up our network bridge and associated settings, allowing the
  container to communicate through this virtual network interface.

#### Permissions and Security:

- Using `sudo` to run the Deno script ensures that we have the necessary permissions to modify system files, which is
  essential since `config.json` affects how the container interacts at the OS level.
- `Deno.run` with `-A` flag grants the script all permissions, which while convenient, should be used cautiously. In a
  production environment, it's advisable to restrict permissions to the minimum required.

### Internet Connectivity Setup

To allow a container to access the internet, we need to define rules that will handle the way data packets are forwarded
between the container and external networks. This involves adjusting the iptables rules and employing Network Address
Translation (NAT).

#### Setup

Here's a detailed breakdown of the iptables configuration:

```sh
ACTIVE_INTERFACE=$(ip route show default | awk '/default/ {print $5}')
sudo iptables -A FORWARD -o $ACTIVE_INTERFACE -i veth-host -j ACCEPT && \
sudo iptables -A FORWARD -i $ACTIVE_INTERFACE -o veth-host -j ACCEPT && \
sudo iptables -t nat -A POSTROUTING -s 192.168.10.1/24 -o $ACTIVE_INTERFACE -j MASQUERADE
```

- `sudo iptables -A FORWARD -o $ACTIVE_INTERFACE -i veth-host -j ACCEPT`: This rule allows forwarding of packets from the external
  network interface (`$ACTIVE_INTERFACE`) to the virtual ethernet interface (`veth-host`) connected to the container.
- `sudo iptables -A FORWARD -i $ACTIVE_INTERFACE -o veth-host -j ACCEPT`: This rule allows forwarding of packets in the opposite
  direction—from the virtual ethernet interface back to the external network.
- `sudo iptables -t nat -A POSTROUTING -s 192.168.10.1/24 -o $ACTIVE_INTERFACE -j MASQUERADE`: This rule enables NAT for the subnet
  used by the container. It masks the container's internal IP address, making outbound connections appear as though they
  originate from the host's IP address, which allows for Internet connectivity.

###### Cleanup

To remove these settings (for example, when shutting down the container or reconfiguring the network), you would issue
the following commands:

```sh
ACTIVE_INTERFACE=$(ip route show default | awk '/default/ {print $5}')
sudo iptables -D FORWARD -o $ACTIVE_INTERFACE -i veth-host -j ACCEPT && \
sudo iptables -D FORWARD -i $ACTIVE_INTERFACE -o veth-host -j ACCEPT && \
sudo iptables -t nat -D POSTROUTING -s 192.168.10.1/24 -o $ACTIVE_INTERFACE -j MASQUERADE
```

These commands delete the previously set iptables rules, effectively cutting off the container's access to the internet
and stopping the forwarding of packets, thereby restoring the original network settings.

#### Nameserver Setup

Proper DNS configuration is crucial for enabling the container to resolve domain names to IP addresses. Below is the
setup to configure DNS services:

```sh
cat <<END | sed 's/^[ \t]*//' | sudo tee ./workspace/bundles/alpine:latest/rootfs/etc/resolv.conf > /dev/null
    # IPv4 dns
    nameserver 1.1.1.1
    nameserver 8.8.8.8
    # IPv6 dns
    nameserver 2606:4700:4700::1111
END
```

- **IPv4 DNS**: `1.1.1.1` (Cloudflare) and `8.8.8.8` (Google) are set as the primary and secondary DNS servers. These
  are robust, widely used public DNS services.
- **IPv6 DNS**: `2606:4700:4700::1111` is the IPv6 address for Cloudflare's DNS service, providing DNS resolution over
  IPv6.

By directing the container to use these DNS servers, you ensure that it can resolve URLs to IP addresses efficiently and
reliably, which is necessary for most network communications involving domain names.

## Conclusion

RunC provides a powerful, low-level toolkit for managing containers directly, adhering to the Open Container
Initiative (OCI) standards. This approach affords users significant control over container configuration and runtime
management, making it an invaluable tool for advanced users, system administrators, and developers seeking a deep
understanding of container internals.

## Summary

In this guide, we explored the core functionalities and benefits of RunC. We began with the setup of essential tools
like **Skopeo** and **Umoci** for handling container images without the need for a full container runtime like Docker.
We then demonstrated how to pull, unpack, and run a container using RunC, highlighting the direct manipulation and
control over the container execution afforded by RunC.

Further, we delved into setting up networking for the container, both within a local setup and for internet
connectivity, using network namespaces, virtual ethernet interfaces, and IP routing. This was complemented by
configuring the NAT and DNS settings to enable robust network communication for the containers.

This guide also navigated through practical uses and implications of running containers in isolated environments,
emphasizing the importance of security considerations and the potential for detailed network configuration and
troubleshooting.

By mastering RunC and its associated tools, users can achieve a high level of proficiency in container management,
tailor systems to specific needs, and maintain greater control over system resources and security configurations. This
makes RunC not only a tool for direct container management but also a platform for innovation and optimization in
containerized environments.
