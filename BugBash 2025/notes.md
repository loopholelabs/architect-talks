# Commoditize Your CI Compute: Spot Instances Without the Spotty Reliability

## Feedback

- Cut the technical part down (takes by far the most time) - merge the per-system slides maybe?
- Make intro shorter

## Meta

- Welcome everyone
- Lots of lightning talks today
- Lots of talks here so far have been about finding bugs and fixing them
- The talk is titled "Commodizite your CI Compute: Spot Instances without the Spotty Reliability"
- and with it I want to take a look at this topic from a bit of a different perspective - the infrastructure that your CI/CD tooling runs on, especially from a cost perspective
- So without further ado, let's get right into it!

## The Problem Landscape (2 minutes)

- Quick show of hands: Who's is not using spot instances or other forms of ephemeral compute because of the risk and complexity?
- Let me explain the spot instance dilemma.

  - Spot instances are 70-90% cheaper than on-demand,
  - but they can be reclaimed with just 30-120 seconds notice.

  - For stateful workloads like builds, GPU training, fuzzing, or integration tests, this is catastrophic.
  - The fear of interruption forces teams into suboptimal choices.

- Most organizations handle this in one of four ways:

  - They stick with expensive on-demand instances
  - use spots only for short-lived stateless jobs
  - build complex workload-specific checkpointing systems
  - or simply accept occasional failures and waste engineering time on retries.

- The cost of maintaining this status quo is staggering.
- Medium-sized engineering organizations spend hundreds of thousands of dollars annually on CI compute alone.
- Beyond the financial impact, teams are running
  - fewer tests
  - shorter fuzzing sessions
  - and simpler integration scenarios than they should.
- Innovation is being constrained by infrastructure limitations rather than engineering capacity.

## The Cloud Computing Lock-In Problem (1 minute)

- This isn't just about costs - it's about digital sovereignty.
- Cloud providers follow the Law of Complements: they monopolize compute while commoditizing adjacent layers.
- Cloud users have lost the freedom to move workloads as costs and needs change.

- Cloud providers create artificial barriers through proprietary "compute" ecosystems like EC2 to prevent true workload portability.
- What's been missing is a primitive that allows applications to be seamlessly portable between providers with no downtime or reconfiguration.

## Introducing Architect (1.5 minutes)

- That's why we built Architect.
- Architect transforms unreliable spot instances across any cloud into a unified, failure-proof compute fabric for all workloads, including CI/CD.

- Our core insight is this:
- Instead of making individual applications more reliable, we've built a system that makes the entire fleet reliable through intelligent workload migration.

- Architect has three key capabilities:

  - 1.  A unit of compute with cross-provider compatibility: We seamlessly run a single VM image and migrate between AWS, GCP, Azure, or on-prem
  - 2.  Live migration: We can move running workloads between hosts in under 10 seconds
  - 3.  Zero application changes: It works with existing CI/CD pipelines without modifications

- You can run anything -

  - CPU accelerated workloads,
  - thousand-node tests,
  - or regular Docker builds

- on the cheapest available compute, without worrying about interruptions.

## The Technology Stack (3.5 minutes)

- Let me explain the technology that makes this possible.

- PVM: Core tech
  - Cloud providers typically disable hardware virtualization extensions to prevent nested virtualization.
  - PVM with our custom patches breakthrough enables fully accelerated VMs without hardware support.
  - This technology was created by Alibaba and Ant Group, with first patches submitted to the Linux Kernel in early 2024.
  - It's compatible with existing hypervisors like QEMU and Cloud Hypervisor with minimal modifications,
    - and allows running custom kernels,
    - kernel modules,
    - and snapshot/restore functionality on standard cloud instances.
    - This makes VMs truly portable across cloud providers without reconfiguration or special permissions.
- Drafter: Compute migration
  - Is the unit of compute
  - Has CPU templates to make every host look the same to the guest even if the CPUs have different feature sets
  - Based on modified version of Firecracker that tracks changes continously
  - VSock-based system for talking with the guest from the host and vice versa (makes talking so services in VMs as easy as talking with containers)
- Silo: Disk migrations
  - Migrates the actual data between the hosts
  - Continously tracks any changes that the guest makes and learns about usage patterns
  - Creates continuous backups and syncs them to S3 as the VM is running
  - When it comes time to migrate from source to destination, it can use the usage patterns it learned about to sync the chunks that are most needed at the destination first to prioritize those chunks over the P2P connection
- Conduit: Network migrations
  - Usually, when migrating from A to B the IP address changes
  - Connections would break
  - Conduit fixes this issue by migrating the connections themselves from A to B
  - Even if there are in-flight packets as the VM is being migrated it will not break
  - True network migrations - after resuming on the destination, the source can be shut down and traffic doesn't need to be routed through the source host anymore
  - Written in eBPF, so works for any L3 protocols (UDP & TCP) and runs on the network card, so it's very fast
  - No changes needed to applications or protocols, runs even with proprietary apps like Minecraft servers
- Mirage: GPU Migrations
  - Migrate GPU workloads across hosts without resetting CUDA contexts.
  - ML training jobs can continue exactly where they left off after being migrated to a different physical host, even on different instance types.

## The Migration Policy Engine (1 minute)

- All of this is orchestrated by our intelligent policy engine, which determines when and where to migrate workloads in real-time.
  - It can automatically evacuate workloads from instances receiving termination notices,
  - move workloads to cheaper instances as they become available,
  - redistribute based on observed resource utilization,
  - and maintain redundancy across zones or regions for critical workloads.
  - The policy engine is cloud-agnostic, enabling unified management across platforms.

## The Demo (2 minutes)

- To show how these migrations work, I'll do a live migration of the TigerBeetle fuzzing tests as they are running.
- This is what would usually run in a GitHub action based on Architect's CI/CD product (show an example GitHub action), but for this demo I'll run it manually so that we can speed up the process - this is a lighting talk after all.
- I'll spin up the fuzzer, then live migrate the VM that's running the fuzzer between GCP, AWS and Azure
- OCI image/VM we're running is public to verify the demo

## Conclusion (30 seconds)

- Architect eliminates the cost excuse for not fixing bugs.
- With 90% lower compute costs,

  - you can run 10x more tests,
  - find more bugs,
  - and build more reliable software.

- We're launching our public beta soon - our first integration is a GitHub actions runner powered by our technology.
- Visit architect.io to join the waitlist or find me after for a deeper technical discussion.

- The future of CI isn't about choosing between cost and reliability - it's about having both, on your terms, across any infrastructure.
