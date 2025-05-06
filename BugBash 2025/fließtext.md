# Fließtext

## Commoditize Your CI Compute: Spot Instances Without the Spotty Reliability

### INTRODUCTION [30 seconds]

Hello everyone! I'm Felicitas—Fel for short—from Loophole Labs.

We've heard fascinating talks today about finding bugs, fixing them, and preventing them in the first place. This talk takes a step back to look at something more fundamental: the actual compute infrastructure running our CI/CD pipelines.

This is a bit of a speedrun—I'm condensing what's normally a 35-minute talk into just 8 minutes, including a live demo! So buckle up!

### THE CURRENT STATE [1 minute]

Quick show of hands: Who here is **not** running their CI/CD on spot instances or ephemeral compute?

[Observe audience]

That's what I thought—and I don't blame you! It makes perfect sense.

Here's the paradox: Spot instances offer incredible benefits:

- 70-90% cost savings compared to on-demand
- Pay only for what you use, when you use it
- More efficient for cloud providers too—they can reclaim capacity when needed

So why isn't everyone using them? Three major reasons:

### THE PROBLEMS [2 minutes]

**The Reclamation Risk**

- 30-120 second termination notices
- Host and all its data gets deleted
- Catastrophic for stateful workloads like builds, GPU training, and integration tests

**The Complexity Tax**
Current "solutions" are all compromises:

1. Don't use spot and overpay for on-demand
2. Limit to stateless jobs (which are only so useful)
3. Build complex, custom checkpointing for each application
4. Accept failures/retries and make everything idempotent

**The Hidden Costs**

- Mid-sized teams waste hundreds of thousands annually on on-demand instances
- Innovation gets constrained:
  - Fewer tests, not running on every commit
  - Shorter or no benchmarks
  - Limited fuzzing sessions
  - Simplified integration test scenarios

**The Lock-In Trap**

- Beyond cost, this is about digital sovereignty
- Cloud providers follow the "Law of Complements"—monopolizing compute while commoditizing the layers below
- Proprietary ecosystems prevent switching between clouds
- No true workload portability between providers

### THE SOLUTION [2 minutes]

This is where Architect comes in. Our core proposition:

A "unified failure-proof compute fabric" based on any number of any type of virtual machine from any cloud.

Three key capabilities make this possible:

1. **Cloud-agnostic VM images and snapshots** — We're not replacing VMs with containers, but creating actual KVM-based VMs that run everywhere

2. **Quick live migration/evacuation** — Faster than the 30-120s termination notice, allowing you to use spot instances like they're regular on-demand instances

3. **Zero code changes** — Support for the entire compute stack (CPU/GPU/disk/network) even during migrations

Our tech stack includes:

- **PVM (Pagetable Virtual Machines)** — A Linux patchset first sent out in 2024 by Ant Group and Alibaba that allows fully accelerated KVM-based VMs on top of VMs without requiring expensive metal instances. Unlike nested virtualization, it doesn't need software emulation or hardware support. Our custom Loophole patches enable cross-cloud VM portability and live migration, including a rewrite of the MMU to speed up forking in the guest VM. Performance overhead is under 10%, far below the cost savings.

- **Drafter (Compute Migration)** — Our open-source (AGPL-3.0) unit of compute that exposes Firecracker via a Go library. It normalizes CPUs across hosts with CPU templates for safe migration between heterogeneous environments, and uses a customized Firecracker fork with continuous tracking of memory changes during runtime. It facilitates guest communication via a VSock-based RPC interface.

- **Silo (Data Migration)** — This open-source component migrates data between hosts, whether it's memory or disk. It continuously tracks changes and learns usage patterns, creating ongoing backups of VM resources and syncing them to S3 asynchronously while the VM runs. When migration time comes, it prioritizes the most-needed chunks first over P2P connection based on learned patterns.

- **Conduit (Network Migration)** — Solves the problem of IP address changes breaking connections during migration. Written in eBPF to run directly on network cards, it supports true network migrations for any L3 protocols (UDP & TCP), even with proprietary payloads. Both ingress and egress migrations are supported, so downloads won't be interrupted when migrating from one host to another.

- **Mirage (GPU Migration)** — Enables migration of GPU workloads with no application changes required. Works for both inference and training out-of-box. Even if an LLM is mid-response, it continues from exactly where it left off without user interruption. Functions across hosts, clouds, and different GPU types just like our CPU migrations.

- **Architect itself** — The "Migration Policy Engine" that coordinates all components through a simple API. It interfaces with GitHub actions, Kubernetes, CRI-compatible interfaces, and REST APIs. It communicates with cloud providers to create instances and resources as needed, and makes proactive decisions like migrating off instances with spot preemption notices, optimizing for cost across providers, ensuring availability in different clouds/zones, and dynamically relocating based on usage patterns.

### LIVE DEMO [2 minutes]

Let me show you how this works in practice.

[DEMO: Migrate TigerBeetle fuzzing tests between AWS/GCP/Azure while the fuzzer continues running]

- This would typically run via Architect's GitHub action integration
- The OCI image we're running is public so you can verify this isn't smoke and mirrors

### CONCLUSION & CALL TO ACTION [30 seconds]

Our public beta is launching soon, and I'd like to invite you to join our waitlist at architect.io.

Our first integration is a GitHub Actions runner powered by Architect that runs in your cloud account on spot instances. If it gets a preemption notice, it automatically spins up a replacement and migrates to it—completely transparent to end-users.

We're also working on Kubernetes integration and support for more CI/CD providers.

The future of CI/CD isn't about choosing between cost and reliability—it's about having both, on your terms, across any infrastructure.

I'm available all day—come talk to me about technical details or potential integrations for your team!

Thank you!

## Commoditize Your CI Compute: Spot Instances Without the Spotty Reliability

Hello everyone! I'm Felicitas—Fel for short—from Loophole Labs. We've heard fascinating talks today about finding bugs, fixing them, and preventing them in the first place. This talk takes a step back to look at something more fundamental: the actual compute infrastructure running our CI/CD pipelines. This is a bit of a speedrun—I'm condensing what's normally a 35-minute talk into just 8 minutes, including a live demo! So buckle up!

Quick show of hands: Who here is **not** running their CI/CD on spot instances or ephemeral compute? [Observe audience] That's what I thought—and I don't blame you! It makes perfect sense. Here's the paradox: Spot instances offer incredible benefits: 70-90% cost savings compared to on-demand, pay only for what you use, when you use it, and they're more efficient for cloud providers too—they can reclaim capacity when needed.

So why isn't everyone using them? Three major reasons: First, the Reclamation Risk. You get just 30-120 second termination notices, the host and all its data gets deleted, which is catastrophic for stateful workloads like builds, GPU training, and integration tests. Second, the Complexity Tax. Current "solutions" are all compromises: Don't use spot and overpay for on-demand, limit to stateless jobs which are only so useful, build complex custom checkpointing for each application, or accept failures/retries and make everything idempotent. Third, the Hidden Costs. Mid-sized teams waste hundreds of thousands annually on on-demand instances, and innovation gets constrained: fewer tests, not running on every commit, shorter or no benchmarks, limited fuzzing sessions, simplified integration test scenarios.

And finally, the Lock-In Trap. Beyond cost, this is about digital sovereignty. Cloud providers follow the "Law of Complements"—monopolizing compute while commoditizing the layers below. Proprietary ecosystems prevent switching between clouds, and there's no true workload portability between providers.

This is where Architect comes in. Our core proposition: A "unified failure-proof compute fabric" based on any number of any type of virtual machine from any cloud. Three key capabilities make this possible: Cloud-agnostic VM images and snapshots—we're not replacing VMs with containers, but creating actual KVM-based VMs that run everywhere; Quick live migration/evacuation—faster than the 30-120s termination notice, allowing you to use spot instances like they're regular on-demand instances; and Zero code changes—support for the entire compute stack (CPU/GPU/disk/network) even during migrations.

Our tech stack includes PVM (Pagetable Virtual Machines), a Linux patchset first sent out in 2024 by Ant Group and Alibaba that allows fully accelerated KVM-based VMs on top of VMs without requiring expensive metal instances. Unlike nested virtualization, it doesn't need software emulation or hardware support. Our custom Loophole patches enable cross-cloud VM portability and live migration, including a rewrite of the MMU to speed up forking in the guest VM. Performance overhead is under 10%, far below the cost savings.

Drafter handles Compute Migration and is our open-source (AGPL-3.0) unit of compute that exposes Firecracker via a Go library. It normalizes CPUs across hosts with CPU templates for safe migration between heterogeneous environments, and uses a customized Firecracker fork with continuous tracking of memory changes during runtime. It facilitates guest communication via a VSock-based RPC interface.

Silo manages Data Migration and is also open-source. It migrates data between hosts, whether it's memory or disk. It continuously tracks changes and learns usage patterns, creating ongoing backups of VM resources and syncing them to S3 asynchronously while the VM runs. When migration time comes, it prioritizes the most-needed chunks first over P2P connection based on learned patterns.

Conduit solves Network Migration challenges, specifically the problem of IP address changes breaking connections during migration. Written in eBPF to run directly on network cards, it supports true network migrations for any L3 protocols (UDP & TCP), even with proprietary payloads. Both ingress and egress migrations are supported, so downloads won't be interrupted when migrating from one host to another.

Mirage enables GPU Migration, allowing migration of GPU workloads with no application changes required. It works for both inference and training out-of-box. Even if an LLM is mid-response, it continues from exactly where it left off without user interruption. It functions across hosts, clouds, and different GPU types just like our CPU migrations.

Architect itself is the "Migration Policy Engine" that coordinates all components through a simple API. It interfaces with GitHub actions, Kubernetes, CRI-compatible interfaces, and REST APIs. It communicates with cloud providers to create instances and resources as needed, and makes proactive decisions like migrating off instances with spot preemption notices, optimizing for cost across providers, ensuring availability in different clouds/zones, and dynamically relocating based on usage patterns.

Let me show you how this works in practice. [DEMO: Migrate TigerBeetle fuzzing tests between AWS/GCP/Azure while the fuzzer continues running] This would typically run via Architect's GitHub action integration. The OCI image we're running is public so you can verify this isn't smoke and mirrors.

Our public beta is launching soon, and I'd like to invite you to join our waitlist at architect.io. Our first integration is a GitHub Actions runner powered by Architect that runs in your cloud account on spot instances. If it gets a preemption notice, it automatically spins up a replacement and migrates to it—completely transparent to end-users. We're also working on Kubernetes integration and support for more CI/CD providers.

The future of CI/CD isn't about choosing between cost and reliability—it's about having both, on your terms, across any infrastructure. I'm available all day—come talk to me about technical details or potential integrations for your team! Thank you!

## Commoditize Your CI Compute: Spot Instances Without the Spotty Reliability

[SLIDE 1 - TITLE]

Hello everyone! I'm **Felicitas**—Fel for short—from **Loophole Labs**. We've heard fascinating talks today about finding bugs, fixing them, and preventing them in the first place. This talk takes a step back to look at something more **fundamental**: the actual **compute infrastructure** running our CI/CD pipelines. This is a bit of a **speedrun**—I'm condensing what's normally a 35-minute talk into just 8 minutes, including a live demo! So buckle up!

[SLIDE 2 - AUDIENCE QUESTION]

Quick show of hands: Who here is **not** running their CI/CD on spot instances or ephemeral compute? [PAUSE] That's what I thought—and I don't blame you! It makes perfect sense. Here's the **paradox**: Spot instances offer **incredible benefits**: **70-90% cost savings** compared to on-demand, pay only for what you use, when you use it, and they're more efficient for cloud providers too—they can reclaim capacity when needed.

[SLIDE 3 - PROBLEMS]

So **why** isn't everyone using them? **Three major reasons**: First, the **Reclamation Risk**. You get just 30-120 second termination notices, the host and **all its data gets deleted**, which is **catastrophic** for stateful workloads like builds, GPU training, and integration tests.

Second, the **Complexity Tax**. Current "solutions" are all compromises: Don't use spot and **overpay** for on-demand, limit to stateless jobs which are only so useful, build complex custom checkpointing for each application, or accept failures/retries and make everything idempotent.

Third, the **Hidden Costs**. Mid-sized teams waste **hundreds of thousands annually** on on-demand instances, and innovation gets constrained: **fewer tests**, not running on every commit, shorter or no benchmarks, limited fuzzing sessions, simplified integration test scenarios.

[SLIDE 4 - LOCK-IN]

And finally, the **Lock-In Trap**. Beyond cost, this is about **digital sovereignty**. Cloud providers follow the "Law of Complements"—monopolizing compute while commoditizing the layers below. **Proprietary ecosystems** prevent switching between clouds, and there's no true workload portability between providers.

[SLIDE 5 - ARCHITECT SOLUTION]

This is where **Architect** comes in. Our core proposition: A "unified **failure-proof compute fabric**" based on any number of any type of virtual machine from any cloud. **Three key capabilities** make this possible: **Cloud-agnostic** VM images and snapshots—we're not replacing VMs with containers, but creating actual KVM-based VMs that run **everywhere**; **Quick live migration**—faster than the 30-120s termination notice, allowing you to use spot instances like they're regular on-demand instances; and **Zero code changes**—support for the entire compute stack (CPU/GPU/disk/network) even during migrations.

[SLIDE 6 - PVM]

Our tech stack includes **PVM** (Pagetable Virtual Machines), a Linux patchset first sent out in 2024 by Ant Group and Alibaba that allows fully accelerated KVM-based VMs on top of VMs **without** requiring expensive metal instances. [BREATHE] Unlike nested virtualization, it doesn't need software emulation or hardware support. Our custom Loophole patches enable **cross-cloud VM portability** and live migration, including a rewrite of the MMU to speed up forking in the guest VM. Performance overhead is **under 10%**, far below the cost savings.

[SLIDE 7 - DRAFTER]

**Drafter** handles Compute Migration and is our open-source (AGPL-3.0) unit of compute that exposes Firecracker via a Go library. It **normalizes CPUs** across hosts with CPU templates for safe migration between heterogeneous environments, and uses a customized Firecracker fork with **continuous tracking** of memory changes during runtime. It facilitates guest communication via a VSock-based RPC interface.

[SLIDE 8 - SILO]

**Silo** manages Data Migration and is also open-source. It migrates data between hosts, whether it's memory or disk. It **continuously tracks changes** and learns usage patterns, creating ongoing backups of VM resources and syncing them to S3 asynchronously while the VM runs. [BREATHE] When migration time comes, it **prioritizes** the most-needed chunks first over P2P connection based on learned patterns.

[SLIDE 9 - CONDUIT]

**Conduit** solves Network Migration challenges, specifically the problem of IP address changes breaking connections during migration. Written in **eBPF** to run directly on network cards, it supports true network migrations for any L3 protocols (UDP & TCP), even with proprietary payloads. Both **ingress** and **egress** migrations are supported, so downloads won't be interrupted when migrating from one host to another.

[SLIDE 10 - MIRAGE]

**Mirage** enables GPU Migration, allowing migration of GPU workloads with **no application changes** required. It works for both inference and training out-of-box. Even if an LLM is mid-response, it continues from exactly where it left off without user interruption. It functions across hosts, clouds, and **different GPU types** just like our CPU migrations.

[SLIDE 11 - ARCHITECT ENGINE]

**Architect** itself is the "Migration Policy Engine" that coordinates all components through a simple API. It interfaces with GitHub actions, Kubernetes, CRI-compatible interfaces, and REST APIs. [BREATHE] It communicates with cloud providers to create instances and resources as needed, and makes **proactive decisions** like migrating off instances with spot preemption notices, optimizing for cost across providers, ensuring availability in different clouds/zones, and dynamically relocating based on usage patterns.

[SLIDE 12 - DEMO]

Let me **show you** how this works in practice. [DEMO: Migrate TigerBeetle fuzzing tests between AWS/GCP/Azure while the fuzzer continues running] This would typically run via Architect's GitHub action integration. The OCI image we're running is public so you can verify this isn't smoke and mirrors.

[SLIDE 13 - CALL TO ACTION]

Our public beta is launching soon, and I'd like to invite you to join our **waitlist** at architect.io. Our first integration is a GitHub Actions runner powered by Architect that runs in your cloud account on spot instances. If it gets a preemption notice, it **automatically** spins up a replacement and migrates to it—completely transparent to end-users. We're also working on Kubernetes integration and support for more CI/CD providers.

[SLIDE 14 - CONCLUSION]

The future of CI/CD isn't about choosing between cost and reliability—it's about having **both**, on your terms, across **any infrastructure**. I'm available all day—come talk to me about technical details or potential integrations for your team! **Thank you!**

# Commoditize Your CI Compute: Spot Instances Without the Spotty Reliability

[SLIDE 1 - TITLE]

Hello everyone! I'm **Felicitas** - Fel for short - from **Loophole Labs**. We've heard fascinating talks in these past few days about finding bugs, fixing them, and writing more reliable software in general. This talk takes a step back to look at something more **fundamental**: the actual **compute infrastructure** running our CI/CD pipelines. This is a bit of a **speedrun** - I'm condensing what's normally a 35-minute talk into just 8 minutes, and that's including a live demo! So buckle up!

[SLIDE 2 - AUDIENCE QUESTION]

Quick show of hands: Who here is **not** running their CI/CD on spot instances or ephemeral compute? [PAUSE] That's what I thought - and I don't blame you! It makes perfect sense. But it seems strange at first, right? Spot instances offer **lots of benefits**: **70-90% cost savings** compared to on-demand, **pay only for what you use**, when you use them, and they're **more efficient for cloud providers** too - they can reclaim capacity when needed.

[SLIDE 3 - PROBLEMS]

So **why** isn't everyone using them? **To us, there are two major reasons**: **First**, the **Reclamation Risk**. You get just 30-120 second termination notices after which the host and **all its data gets deleted**, which is **catastrophic** for stateful workloads like long-running builds, GPU training, and integration tests.

**Second**, the **Complexity Tax**. Current "solutions" are all compromises: **Don't use spot** and **overpay** for on-demand, **limit to stateless jobs** which are only so useful, **build complex custom checkpointing** for each application, or **accept failures/retries** and make everything idempotent.

[SLIDE 4 - CONSEQUENCES]

This has real consequences. Mid-sized teams waste **hundreds of thousands annually** on on-demand instances (even at Loophole we spend a few grand per month), and those cost constraints lead to real issues: **fewer tests are being written**, **they aren't running on every commit**, **there are shorter or no benchmarks at all**, **only limited fuzzing sessions**, **or only simplified integration test scenarios are used because they would otherwise be too expensive**.

And finally, the **Lock-In Trap**. Beyond just cost, this is about **digital sovereignty**. Cloud providers themselves follow the "Law of Complements" - monopolizing compute while commoditizing the layers below, but as a user of those cloud providers you can't do that yourself. **Proprietary ecosystems** prevent switching between clouds, even for something as "simple" as compute/VPSes, and there's no true workload portability between providers.

[SLIDE 5 - ARCHITECT SOLUTION]

This is where **Architect** comes in. The core goal is: A "unified **failure-proof compute fabric**" based on any number of virtual machines from any cloud provider. **Three key capabilities** make this possible: **First**, **Cloud-agnostic** VM images and snapshots - we're not replacing VMs with containers, but creating actual KVM-based VMs that run **everywhere**. It's **not a lossy abstraction** - VMs stay VMs. **Second**, **Quick live migration** - faster than the 30-120s termination notice, allowing you to use spot instances as if they were regular on-demand instances; and **Third**, **Zero code changes** are necessary - there is support for the entire compute stack (CPU/GPU/disk/network) even during migrations.

[SLIDE 6 - PVM]

Our tech stack is based **PVM** (Pagetable Virtual Machine), a Linux patchset first sent out in 2024 by Ant Group and Alibaba that allows fully accelerated KVM-based VMs on top of VMs **without** requiring expensive metal instances. [BREATHE] Unlike nested virtualization, it doesn't need software emulation or hardware support - no consent or buy-in from the cloud providers is needed, it works anywhere that Linux runs. Our custom Loophole patches enable **cross-cloud VM portability** and live migration with PVM, including a rewrite of the MMU to speed up forking in the guest VM. Performance overhead is **under 10%**, far below the cost savings you get.

[SLIDE 7 - DRAFTER]

**Drafter** handles Compute Migration and is our open-source (AGPL-3.0) unit of compute that exposes Firecracker via a Go library. It **normalizes CPUs** across hosts with CPU templates for safe migration between heterogeneous environments, and uses a customized Firecracker fork with **continuous tracking** of memory changes during runtime. It also enables efficient zero-copy **guest-host communication** via a VSock-based RPC interface.

[SLIDE 8 - SILO]

**Silo** manages Data Migration and is also open-source (AGPL-3.0). It migrates data between hosts, whether it's memory or disk. It **continuously tracks changes** and learns usage patterns, creating continous backups of VM resources and syncing them to S3 asynchronously while the VM runs. [BREATHE] When migration time comes, it **prioritizes** the most-needed chunks first over P2P connection based on learned patterns, and fetches the **last-needed** chunks from S3 so that the source host can be shut down ASAP.

[SLIDE 9 - CONDUIT]

**Conduit** solves Network Migration, specifically the problem of the IP address changing after you've migrated from source to destination. Usually that breaks connections. Written in **eBPF** to run directly on network cards, it supports true network migrations for any L3 protocols (UDP & TCP), even with proprietary payloads. Both **ingress** and **egress** migrations are supported, so downloads that the guest has started won't be interrupted either when migrating from one host to another.

[SLIDE 10 - MIRAGE]

**Mirage** is an experimental component that enables full GPU Migration, allowing migration of GPU workloads with **no application changes** required. It works for both inference and training out-of-box. Even if an LLM is mid-response, it continues from exactly where it left off without user interruption. It functions across hosts, clouds, and **different GPU types** just like our CPU migrations. It's out of scope for this talk and still in closed beta, but if you're interested in this talk to me afterwards or reach out to me and I'd love to talk more about it.

[SLIDE 11 - ARCHITECT ENGINE]

**Architect** is the "Migration Policy Engine" that orchestrates all components through a simple API, and acts as a kind of conductor for compute, disk, and GPU migrations. It integrates with **GitHub actions**, **Kubernetes**, **CRI-compatible interfaces**, and exposes it's own **REST API**. It also integrates with cloud providers to provision resources on demand. Architect makes **proactive decisions** by **rapidly migrating off instances** when receiving spot preemption notices or maintenance alerts, it can **optimize costs** across different cloud providers by automatically moving to cheaper options without downtime ("cross-cloud spot arbitrage"), can **make systems highly available** through automatic redundancy across multiple clouds and availability zones, and can **dynamically relocate** VM based on changing usage patterns such as user activity with time zones (e.g. if the East Coast goes to sleep) to move a system closer to users.

[SLIDE 12 - DEMO]

Let me **show you** how this works in practice. [DEMO: Migrate TigerBeetle fuzzing tests between AWS/GCP/Azure while the fuzzer continues running] This would typically run via Architect's GitHub action integration. The OCI image we're running **is public** too so you can verify this isn't smoke and mirrors.

[SLIDE 13 - CALL TO ACTION]

Our public beta is launching soon, and I'd like to invite you to join our **waitlist** at architect.io. Our first integration is a GitHub Actions runner powered by Architect that runs in your cloud account on spot instances. If it gets a preemption notice, it **automatically** spins up a replacement and migrates to it - completely transparent to end-users. We're also working on Kubernetes integration and support for more CI/CD providers.

[SLIDE 14 - CONCLUSION]

The future of CI/CD isn't about choosing between cost and reliability - it's about having **both**, on your terms, across **any infrastructure**. I'm available all day - just come talk to me about technical details, integration points or anything else. **Thank you!**
