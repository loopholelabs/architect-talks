# Commoditize Your CI Compute: Spot Instances Without the Spotty Reliability

Intro/Title

---

Are you running your CI/CD on spot instances?

---

Why are you not running your CI/CD on spot instances?

- 70-90% cheaper
- Pay only for what you use
- Better for cloud providers, too

---

Why isn't everyone using Spot?

30-120s termination notice (in big font/as fact)

---

What are the existing "solutions"?

- Don't use spot and overpay
- Only run stateless jobs
- Build complex custom checkpointing
- Accept failures/retries

---

Consequences of not being able to use Spot

- Hundreds of thousands of dollars lost anually
- Fewer tests written
- Tests aren't running on every commit
- Short or no benchmarks
- No or little fuzzing

---

Gwern Laws of Tech: Commoditize Your Complement (screenshot from https://gwern.net/complement)

---

Unified failure-proof compute fabric (in big font)

---

How Architect's capabilities make this possible

- Cloud-agnostic VM images and snapshots
- Very fast live migration
- Zero code changes required

---

PVM

- First posted in 2024
- Run VMs on top of VMs without nested virt or emulation
- Custom Loophole Labs patches allow using it for live migration
- Performance overhead is well below 10%

---

Drafter (picture of GitHub repo README)

- Hypervisor with first-class live migration support
- Normalizes CPUs across hosts
- Uses our fork of the well-tested Firecracker internally

---

Silo (picture of GitHub repo README)

- Data migration system
- Continously tracks changes & learns access patterns
- Continously backs up to S3 as VM is running
- Streams important chunks over P2P connection, less important chunks can be fetched from S3

---

Conduit

- Supports true network migrations during a migration
- Written in eBPF, runs at line speed & runs on network card
- Works on L3
- Supports migrating ingress & egress traffic, it looks like nothing happened to the VM and the client

---

Mirage

- Enables live migration of GPU workloads
- No need for application changes
- Supports inference and training
- Can migrate between hosts and different GPUs

---

Architect itself

- Orchestrates Drafter, Silo, Conduit and Mirage
- Exposes GitHub actions, Kubernetes and REST API integrations
- Provisions actual instances from cloud providers

- Rapidly migrating off instances when receiving a spot pre-emption notice
- Optimizes cost by automatically migrating to cheaper options without downtime ("cross-cloud arbitrage")
- Dynamically migrates the entire fleet to a different physical location if access patterns change

---

Demo

---

Outro/CTA
