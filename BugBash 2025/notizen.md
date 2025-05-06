# Commoditize Your CI Compute: Spot Instances Without the Spotty Reliability

(Intro/Title)

Hi everyone! I'm **Felicitas** - Fel for short - from **Loophole Labs**. We've heard lots of fascinating talks in these past few days about finding bugs, fixing them, and writing more reliable software in general. This talk takes a bit of a step back to look at something more **fundamental**: the actual **compute infrastructure** that we run our CI/CD pipelines on. This is a bit of a **speedrun** - I'm condensing what's normally a 35-minute talk into just 8 minutes, including a live demo - so buckle up!

(Are you running your CI/CD on spot instances?)

I'd like to start with a quick show of hands: Who here is running their CI/CD on spot instances or other forms of ephemeral compute right now? [PAUSE] That's what I thought - and I don't blame you! It makes perfect sense. But it does seems a bit strange at first, right?

(Why are you not running your CI/CD on spot instances?)

Spot instances offer **lots of benefits** after all: They save **70-90% of costs** compared to on-demand, you **pay only for what you use**, and when you use them, they're even **more efficient for cloud providers** too since they can reclaim capacity when needed.

(Why isn't everyone using Spot?)

So **why** isn't everyone using them? To us, the main reason is that you get just a **30-120 second termination notices** - after that the host and **all its data gets deleted**, which is **catastrophic** for stateful workloads like long-running builds, GPU training, and integration tests.

(What are the existing "solutions"?)

It's undeniable that the current "solutions" to get around this are all compromises: You just **don't use spot** and **overpay** for on-demand, **limit yourself to stateless jobs** which are only so useful, **build complex custom checkpointing** for each application (which is popular in the AI space), or **accept failures/retries** and make everything idempotent, which requires lots of engineering work.

(Consequences of not being able to use Spot)

This has real consequences. Even medium-sized teams can waste **hundreds of thousands annually** on on-demand instances (even a startup like us spends a few grand per month), and those cost constraints lead to real issues: **fewer tests are being written**, **tests aren't running on every commit**, **there are short benchmarks or no benchmarks at all** or **only very few fuzzing sessions are running**.

(Gwern Laws of Tech: Commoditize Your Complement (screenshot from https://gwern.net/complement))

And finally, this goes beyond just cost, it's also about **digital sovereignty**. Cloud providers themselves follow the "Law of Complements" - monopolizing compute while commoditizing the layers below, but as a user of those cloud providers you can't do that yourself. **Proprietary ecosystems** prevent switching between clouds, even for something as "simple" as compute/VPSes, and there's no true workload portability between providers.

(Unified failure-proof compute fabric)

This is where **Architect** comes in - it allows you to commoditize your completent, the cloud provider. The core goal is to create a "unified **failure-proof compute fabric**" based on spot instances from any number of cloud providers.

(How Architect's capabilities make this possible)

**Three key capabilities** make this possible: **First**, **cloud-agnostic** VM images and snapshots - we're not replacing VMs with containers, but creating actual KVM-based VMs that run **everywhere**. It's **not a lossy abstraction** - VMs stay VMs. **Second**, **Quick live migration** - faster than the 30-120s termination notice, allowing you to use spot instances as if they were regular on-demand instances; and **third**, **zero code changes** are necessary - there is support for the entire compute stack (CPU/GPU/disk/network) even as a migration is happening, nothing ever goes away or errors out.

(PVM)

Our tech stack that accomplishes this is based **PVM** (Pagetable Virtual Machine), a new work-in-progress Linux patchset first posted by Alibaba and Ant Group in 2024 that allows fully accelerated KVM-based VMs on top of VMs **without** requiring expensive metal instances. [BREATHE] Unlike nested virtualization, it doesn't need software emulation or hardware support - no consent or buy-in from the cloud providers is needed, it works anywhere that Linux runs. Our custom Loophole patches extend PVM to work consistently across heteregenous infrastructure and work with live migration, including a rewrite of the MMU to speed up forking in the guest VM. Performance overhead is **under 10%**, far below the cost savings you get.

(Drafter)

**Drafter** handles compute migration and is our open-source (AGPL-3.0) unit of compute with first-class live migration support. It's written in Go and based on Firecracker, which allows it to **normalize CPUs** across hosts for safe migrations between heterogeneous environments. We use it with our customized Firecracker fork that supports **continuous tracking** of memory changes during runtime.

(Silo)

**Silo** manages data migration and is also open-source (AGPL-3.0). It migrates data between hosts, whether that's memory, disk or any other VM state. It **continuously tracks changes** and learns access patterns, which it then uses to create continous backups of VMs that are synced to S3 asynchronously while the VM runs. [BREATHE] When the actual migration happens, it **prioritizes** the most-needed chunks first over a P2P connection based on those learned patterns, and fetches the **last-needed** chunks from S3 so that the source host can be shut down ASAP.

(Conduit)

**Conduit** solves the problem of the IP address changing after you've migrated from source to destination. Usually that breaks connections. Conduit is written in **eBPF**, which means it runs directly on network cards, which makes it run at essentially line speed. It supports true network migrations for any L3 protocols (UDP & TCP), even with proprietary protocols that we can't control (e.g. the Minecraft server). Both **ingress** and **egress** connection migrations are supported, so downloads that the guest has started before or during a migration for example won't be interrupted either when migrating from one host to another. It's a bit out of scope for this talk, but I'll link a different, more in-depth talk I did on this at the end.

(Mirage)

**Mirage** is an experimental component that enables live migration of GPU workloads with **no application changes** required. It works for **both inference and training** out-of-box. Even if an LLM is mid-response, it continues from exactly where it left off without user interruption. It functions across hosts, clouds, and **different GPU types** just like our CPU migrations. It's out of scope for this talk and still in closed beta, but if you're interested in this, talk to me afterwards, I'd love to talk more about it.

(Architect the orchestrator)

And finally, **Architect** what we like to call the "Migration Policy Engine". It acts as a kind of orchestrator and exposes a simple, unified API that ties compute, disk, and GPU migrations together.

(Architect itself)

It integrates with **GitHub actions**, **Kubernetes** and exposes it's own **REST API**. It ofc also integrates with the cloud providers to provision the actual pyhsical instances.

Architect makes **proactive decisions** by **rapidly migrating off instances** when it receives a spot preemption notice or a maintenance alert, it can **optimize costs** across different cloud providers by automatically moving to cheaper options without downtime (what we like to call "cross-cloud spot arbitrage"), and it can **dynamically relocate** an entire fleet of hosts without downtime if access patterns change. For example, if the East Coast goes to sleep, it can live migrate the entire system to the West Coast to reduce latency.

(Demo)

Enough theory though, I want to show you how this actually works in practice. For that, I want to migrate the TigerBeetle fuzzing tests between AWS/GCP/Azure while the fuzzer continues running as though nothing happened. This would typically run via Architect's GitHub action integration, but to speed things up (we only have a minute remaining) I'll do it manually. The OCI image we're running **is public** too, so you can verify this isn't smoke and mirrors. [DEMO]

(Outro/CTA)

If you want to try this out - our public beta is only a few weeks away now, and there is a waitlist at architect.io that you can sign up for. Our **first integration is a GitHub Actions runner** powered by Architect that runs **in your cloud account on spot instances**. If it gets a preemption notice, it **automatically** spins up a replacement and migrates to it - completely transparent to end-users. We also have a working Kubernetes integration and support for more CI/CD providers coming.

And with that I'd like to thank you for listening. I hope that this talk has shown that the future of CI/CD isn't about choosing between cost and reliability anymore - it's about having **both**, on your terms, across **any infrastructure**. And if you want to know more, I'll be here all day, and everything I've mentioned will be in the link behind this QR code!
