# Building Reliable Cross-Cloud Kubernetes Clusters on Spot Instances with Drafter and PVM

> Felicitas Pojtinger, Loophole Labs

Building Kubernetes clusters that span across multiple cloud providers prevents vendor lock-in and offers flexibility. Using spot instances can further cut costs by up to 90%, but they can terminate with only 30 seconds' notice. Traditionally, migrating VMs across cloud providers and CPUs to mitigate this has been challenging due to hardware constraints. PVM (Pagetable Virtual Machine) is an experimental kernel technology that changes this by enabling KVM without hardware assistance or emulation. Using the research paper, this session will explain how PVM works and how the open-source Drafter and Firecracker projects can use it to migrate VMs between cloud providers. The session includes a live demo of running Kubernetes components like the Kubelet, CRI, CSI and CNI inside VMs and migrating them in a heterogeneous EC2, GCP, and Azure environment. This allows evacuating a Kubernetes node and network without downtime if a spot instance is terminated or if another provider is cheaper.

## Presentation Intro

> TODO

## Speaker and Company Intro

> TODO

## Idea and Motivation

- **Law of Complements**: The concept that layers of a technology stack often aim to monopolize, while simultaneously pushing adjacent layers into highly competitive, commoditized markets.
  - **Examples**: IBM supports Linux because it complements its hardware; car manufacturers advocate for free, commoditized roads since they complement their product.
- **Lock-In Levels as Trade Restrictions**: Cloud software users ideally want abstract, general "compute" resources, yet cloud providers create barriers to make "compute" proprietary, branding it as AWS Compute, EC2 Compute, etc.
- **Comparative Advantage without Restrictions**: Removing these restrictions can benefit customers by providing truly fungible compute resources, fostering a free cloud market.
  - In this market, spot instances could be viable, as switching providers would be risk-free and downtime-free.
- **Digital Sovereignty**: For cloud users and operators, the cloud itself could be a commoditized complement, allowing them to reclaim digital autonomy.
  - Users could switch between providers freely as costs change.
- **Challenges with Current Implementations**:
  - **VM Limitations**: Running VMs on top of most cloud providers requires costly bare-metal instances, which also have drawbacks like long provisioning times. This limitation prevents cloud-agnostic application definitions that provide users with as much control as the cloud providers themselves.
  - **Lack of Portability**: Applications can't be seamlessly moved between providers. Even portable code often results in downtime.
- **Imagining a Solution**:
  - What if any virtual cloud server could function as if it were bare metal, supporting VM creation and migration?
  - Applications could move with the user or closer to end-users, optimizing latency.
  - Instead of moving data to users, users could move applications closer, processing data in secure data centers and only transferring results.
  - Switching cloud providers could be as simple as migrating VMs—no downtime or extensive reconfiguration required.
  - Consistent environments eliminate the need for adjusting to provider-specific quirks (e.g., custom Linux distros with differing configurations).
- **Flexible Application Management**:
  - Applications can be temporarily moved off-site for hardware updates and moved back seamlessly.
  - Continuous mirroring allows for on-demand deployment in case of issues.
  - Integration with secure enclaves/confidential computing enables deployment to untrusted or high-security environments (e.g., banks).
  - Applications could be transferred from remote machines to local systems for debugging (e.g., CI/CD job troubleshooting).
  - Public cloud resources could be leveraged during peak demand, with a fallback to on-prem capacity.
  - Idle on-prem server capacity could be rented out for benchmarking, CI/CD, and more.
  - Cost savings by utilizing spot instances over more expensive on-demand instances—potentially reducing costs by 90%.

## KVM, PVM, and Their Relationship

- **KVM Overview**:

  - KVM (Kernel-based Virtual Machine) is a low-level Linux component providing a vendor-agnostic interface for CPU virtualization. It typically relies on vendor-specific, hardware-based virtualization extensions to isolate guest VMs securely from one another.
  - Many hypervisors, including QEMU, Cloud Hypervisor, and (more recently) VMware, use KVM for CPU virtualization while implementing virtual devices in userspace, often through `virtio`.
  - KVM can be implemented through various means, not just with hardware extensions. For example, Intel's `kvm_intel` module leverages Intel's VT-d, and AMD's `kvm_amd` uses AMD-V technology to interface with respective hardware virtualization features.

- **Cloud Provider Limitations**:

  - Cloud providers generally disable hardware virtualization extensions (such as Intel VT-d and AMD-V) to prevent nested virtualization on their instances. This is the default setting on platforms like EC2, GCP, and Azure.
  - Even when hardware virtualization is enabled (e.g., on certain GCP instances), virtualizing the virtualization instructions themselves results in significant latency, scheduling issues, and spikes, as seen in latency studies (see latency spike graph from the PVM paper).
  - Without hardware support, virtualizing the CPU requires full emulation (e.g., via QEMU), which is very slow and resource-intensive. This effectively restricts VM usage on cloud providers without bare-metal access, hindering the creation of compute units equivalent to the cloud provider’s native VM instances.

- **Introducing PVM**:

  - PVM (Portable Virtual Machine) is an emerging, experimental virtualization technology designed to enable fully accelerated VMs without hardware support. With PVM, virtual machines can run on cloud provider instances without requiring hardware virtualization extensions, and thus, without needing explicit consent or configuration changes from the provider.
  - PVM implements nearly all of KVM’s functionality, making it compatible with most hypervisors (e.g., QEMU) with minimal modifications.
  - Since PVM behaves like KVM, users can run custom guest kernels on public cloud providers, load custom kernel modules, and enable snapshot/restore functionality, providing more flexibility than traditional cloud instances.

- **Benefits of PVM**:
  - **Portability**: With PVM, VMs and their snapshots and images—become portable and can be migrated across different cloud providers without reconfiguration.
  - **Feature Control**: PVM allows selective enabling/disabling of CPU features, supporting a custom and flexible virtual environment.
  - **Cloud Flexibility**: PVM enables fully-featured VMs on public cloud platforms, bridging the gap between secure container platforms and traditional VMs.
- **Implementation and Development**:
  - PVM is developed by Alibaba and Ant Group, with the first RFC patches submitted to the Linux Kernel Mailing List (LKML) in early 2024.
  - Primarily designed for secure container solutions like Kata Containers, PVM also enables hypervisors like QEMU and Cloud Hypervisor to run traditional VMs, broadening its application.
  - PVM is implemented as a kernel module but currently requires non-mainlined kernel patches for full functionality.

## Firecracker, CPU Templates & `MAP_SHARED`/`msync` for Continuous Snapshotting

- **Firecracker Overview**:
  - Firecracker is a lightweight hypervisor developed by Amazon, designed for creating "microVMs" that start and stop rapidly. It implements only the essential features/devices needed for VM operation, optimizing speed and efficiency.
  - Written in Rust, Firecracker is generally more hackable than larger, complex hypervisors like QEMU.
- **CPU Templates**:

  - Firecracker supports CPU templates, allowing selective hiding of CPU features from the guest. This is useful for creating consistent clusters across nodes with differing host CPUs.
  - For example, the `T2CL` template makes all Intel CPUs appear as Intel Cascade Lake to the guest, while `T2A` does the same for AMD CPUs, presenting them as AMD Milan CPUs.

- **Snapshot/Restore Functionality**:

  - Firecracker enables full snapshot and restore functionality, allowing a VM’s memory and CPU state to be saved and later restored, effectively "stopping time" from the VM's perspective.
  - Combined with CPU templates, snapshots enable a VM to be paused on one host, saved, and restored on another—even with a different CPU—by matching the lowest common CPU feature set.

- **PVM Integration**:

  - A small patch enables using Firecracker’s snapshot/restore function with PVM by including specific PVM MSRs (Model-Specific Registers).
  - By combining CPU templates, snapshot/restore functionality, and PVM, it becomes feasible to migrate VMs across different cloud providers without needing bare-metal or nested virtualization and without losing state.

- **Challenges with Traditional Snapshotting**:

  - Dumping the entire VM memory and CPU state to a file can be slow and resource-intensive; a 16 GB VM, for example, can take several seconds to dump.
  - Once dumped, the memory file must be transferred to the destination, which is problematic if the VM is on a spot instance that could be shut down with only a 30-120 second warning.

- **Continuous Snapshotting Solution**:
  - To address this, we patched Firecracker for continuous snapshotting, reducing the need for repeated full memory dumps by tracking incremental changes.
  - By default, Firecracker `mmap`s the memory region with `MAP_PRIVATE`, meaning writes go to anonymous memory regions. During snapshotting, these regions are merged with original snapshots to create the final file.
  - Our modification uses `MAP_SHARED`, which writes changes directly to the underlying file as the VM runs, allowing for real-time syncing of memory pages.
  - To force memory changes to disk immediately, we can call `msync`.
  - With this setup, we continuously monitor changes to the file that Firecracker `mmap`ed, enabling real-time tracking of guest memory changes without needing to halt the VM.

## Silo

- To sync data from the old host to a new host when moving a VM, or to quickly start a VM from a remote image, we need to expose a virtual file to Firecracker that contains the memory snapshot, CPU registers, disk, etc.
- As the kernel asynchronously writes back changes to that file, this virtual file should handle the changes and sync them to the destination.
- Since both reads and writes can be tracked, we can perform a "hybrid" migration.
- On the source, the VM starts syncing memory chunks to the destination while running, until the changes per iteration are small enough to stop time, switch to the destination, and resume there.
- The destination continues receiving chunks marked as dirty or changed by the source before time was stopped.
- If a chunk is unavailable on the destination and needed immediately, it’s requested from the source and prioritized.
- Locks ensure no stale data is read.
- Silo implements these virtual files through NBD devices.
- It’s open-source (AGPL-3.0) and includes a high-performance NBD client and server.
- Features an NBD-independent internal storage engine that supports P2P migrations, migrations with a third-party data store as the data plane (e.g., S3), and hybrid approaches where pre-migration is done using S3, but post-migration is P2P for speed.
- Includes both a CLI and a user-friendly library.
- Independent of the transport layer; it can operate over TCP, TLS, UNIX pipes, etc.
- Includes an efficient copy-on-write layer, integrity verification, and helps deduplicate data across multiple sources (e.g., S3).
- Can also mount file systems between the host and guest kernel over VSock with full read-write semantics with a custom kernel module

## Drafter

- Drafter integrates Silo and Firecracker, connecting Silo’s virtual file with Firecracker’s VM resources (memory, CPU registers, disk images, kernel).
- It initiates connections between source and destination, coordinating Firecracker’s VM lifecycle (`msync`, requesting dirty pages, etc.) with Silo.
- "Lightweight VMmotion for the cloud"
- Simplifies VM startup to a single library call in Go, with robust recovery handling and context awareness.
- Includes a full bi-directional VSock-based agent system to notify the guest of host events, execute commands, or export metrics based on `panrpc`.
- Offers all necessary components to build a VM-based software platform:
  - `drafter-nat` and `drafter-forwarder` enable port forwarding from the VM, similar to Docker.
  - `drafter-agent` and `drafter-liveness` enable efficient VSock-based RPC calls between the guest and host.
  - `drafter-snapshotter` allows creating fresh VM snapshots, configurable with CPU templates, memory allocation, and CPU settings.
  - `drafter-packager` creates distributable packages from VM images (“blueprints”) or snapshots (“packages”) and extracts them.
  - `drafter-runner` starts a VM package/snapshot, ideal for local testing or VMs on a single host.
  - `drafter-registry` serves VM packages/snapshots over the network for quick remote VM startups.
  - `drafter-mounter` enables resource sharing (e.g., disk, kernel, memory) between two VMs.
  - `drafter-peer` streams a VM from the network or starts it from a local snapshot, enabling VM migration across hosts.
  - `drafter-terminator` supports migrating a VM to a host without starting it, useful for terminating a P2P migration chain.
- With PVM and Firecracker patches, Drafter runs broadly without needing nested virtualization or additional VMs.

## Demos #1

### Live Migrating Valkey/Redis between Cloud Providers with Drafter

- Moving between EC2, GCP, Azure and Hetzner while clients get disconnected
- This is using the Valkey OCI image as the payload, which is started with `crun`
- Do demo

### Live Migrating a Kubernetes Cluster between Cloud Providers with Drafter

- Demo of moving the k3s cluster with Drafter
- One of the benefits of using VMs is that we can run every application inside of them
- While running Kubernetes & it's CNI, CSI, CRI etc. plugins inside of a container is hard to impossible, it's normal, safe and expected to do so in a VM
- In our case, we've created a VM that runs Kubernetes, and we want to move that specific, universal VM image from EC2 to GCP to Azure and finally to Hetzner
- The entire Kubernetes cluster including all of it's services will get migrated, and it Just Works™
- This is a first since building a Kubernetes cluster with such heterogenous nodes is usually a problem - each cloud provider usually has a slightly different base OS and CPU features, but with VMs, Drafter and PVM the actually same image can be deployed to all of the cloud providers, completely eliminating the issues
- Do demo

## Network Live Migration with Conduit

- So far, the network connection was cut during each migration
- This is because the physical IP address of the Kubernetes cluster/it's nodes change each time the VM gets migrated
- Conduit is another Loophole product (proprietary) that allows us to migrate network connections between nodes
- It's extremely fast (can push 40GB/s), does not decrypt or intercept/proxy/modify your data (it's fully opaque); written in eBPF
- Works for both outbound and inbound connections (so both e.g. the database server your server is connected to, and the user that's connected to your server, stay connected)
- Doesn't require the original host to stay up during migrations - connections actually do migrate to the destination host, e.g. your ping gets lower after migrating

## Connecting Everything together with Architect

- Architect for integrating Conduit, a control plane, API and dashboard, metrics, integrating with creating VMs on public cloud to then run PVM VMs on, listening for spot instance evictions & moving away in time, plus integrations with other services (starting VMs, GitHub actions and other CI/CD providers, migrating Kubernetes pods with a custom CRI)

## Demos #2

### Honorable Mentions

- Mention moving Valkey with Architect (like in last KubeCon)
  - Same VM image as before
  - No change to the application at all
  - By simply using Conduit, the migration happens between the different regions/data centers, without any interruption or downtime
  - This means that we can use Spot instances for pretty much every application - including stateful ones, since they can just move off the VMs before they get destroyed!
- Mention moving the k3s cluster with Architect (with everything pointed at the Conduit ingress)
  - Now we'll do the migration again - this time however, the connections even between the Kubernetes nodes will not be broken during a migration, and the service we've started will also keep on being online the entire time
  - We can even use `kubectl` during the migration to start something new
  - This allows building an entire Kubernetes cluster on spot instances - the kubelet can simply be evacutated before the instance gets deleted, without anyone noticing!

### Moving Kubernetes Pods between Cloud Providers with Architect

- Demo of moving k8s pods with Architect (this is the actual demo)
  - So far, we've moved an entire Kubernetes cluster
  - But that's complicated, and sometimes you really only want to move some pods between clusters
  - We've built an entire Kata Containers-like system that transparently runs pods in any external Kubernetes distribution (EKS etc.) in virtual machines, and allows those pods and their network to be live-migrated
  - There is no need for a CRD anymore - you simply create a deployment and specify where the pod should run and add an annotation
  - When you want to migrate the VM to a different host, simply delete it and re-create it with the same annotation
  - When the VM gets "re-created", it actually gets migrated in from the old source host automatically
  - Like before - thanks to Conduit, there is no downtime, and this works across cloud providers!
  - Do demo

### GitHub Actions CI/CD on Spot Instances with Architect

- Mention CI/CD with Architect
  - One of the best uses of spot instances is for running CI/CD or benchmarking
  - A lot of CPU resources are required here usually, and that gets expensive
  - Sometimes you even just want to run CI/CD on your own infrastructure, but that gets very complicated if you want true isolation/run each of your workloads in a fresh VM that's guaranteed to be compatible with GitHub's own setup
  - Most providers that use public cloud instances schedule to images that are wildly different from the GitHub actions base image, meaning that it will not work by default, or they use Docker instead of VMs which breaks almost all Docker-based workflows
  - With Architect, you can run CI/CD in public cloud, but still get the exact same image that powers GitHub actions
  - It can also run on spot instances, which gives you an instant 90% discount (if you have a $1000/month CI/CD AWS bill, that bill will go down to $100/month) - and if the spot instance gets pre-empted, Architect instantly detects this and moves CI/CD off to a second host without anyone noticing and no downtime
  - This is the first product we're officially making available today, you can head on over to `architect.run`, connect your GitHub & AWS accounts, replace `ubuntu-latest` with `architect-ubuntu-latest`, and you'll schedule workloads using PVM on spot instances!

## Recap

- For you as a developer, operator or company, you want your complement (the cloud) to be a truly fungible, commoditized resource
- PVM allows running a familiar universal compute unit that behaves exactly the same way across all clouds (a VM) - even if the cloud provider doesn't allow nested virtualization and you're not using bare metal
- Silo allows migrating a lot of data between cloud providers very quickly and efficiently
- Drafter can take that new, universal compute unit based on Silo and Firecracker and migrate it between regions, cloud providers and even continents in a very easy way, even if it's as complex as an entire Kubernetes cluster
- Conduit can then also take all inbound and outbound network connections and move them between nodes, between regions and cloud providers, meaning that you can move a VM without any downtime happening along the process
- Architect allows integrating all of these systems together and integrates them with your existing external services, such as your Kubernetes distro of choice, your own in-house orchestrator, or your CI/CD provider, and connects them to your cloud provider and on-prem infrastructure so that it can automatically spin up VMs and migrate them if spot instances get pre-empted

## Acknowledgements

- OSS repo list

## Speaker and Company Outro

> TODO

## Call to Action

- Architect Website
- CTA (signing up for the CI/CD product)
