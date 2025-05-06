# Commoditize Your CI Compute: Spot Instances Without the Spotty Reliability

What if your CI/CD pipeline could run _anything_ - GPU-accelerated fuzzers, 1,000-node integration tests, or your regular Docker builds - on ephemeral spot instances across _any cloud environment_, with zero downtime, while cutting costs by up to 90%? This talk introduces **Architect**, a new stack that turns unreliable spot instances and heterogeneous cloud environments into a unified, failure-proof compute fabric optimized for CI workloads.

Felicitas will demonstrate how **Architect** uses **Drafter** (a compute primitive for live migration), **Silo** (a novel data migration framework), **Conduit** (an eBPF-based TCP/UDP connection migration engine), and **PVM** (Pagetable Virtual Machine) to:

- Automatically evacuate stateful CI workloads (e.g. long-running builds or multi-hour ML training) during spot instance termination using an intelligent policy engine, which orchestrates migrations based on real-time factors like spot pricing, availability, preemption signals, or direct API calls.
- Migrate GPU workloads seamlessly across hosts _without resetting CUDA contexts_ and preserve live network sessions (WebSockets, gRPC streams) via eBPF-driven stateful connection handoffs.
- Make testing more deterministic via Silo's storage layer and Drafter's continous backups, which snapshots consistent VM state to (object) storage at configurable intervals and allows resuming the exact state a bug happened in later

While traditional CI systems avoid spot instances for fear of preemption, Architect exploits their volatility to eliminate the cost excuse for not fixing bugs. With Architect, you can run 10x more fuzz tests, 10x more integration scenarios, and 10x longer simulations for the same price—all on spot instances, all without fear.
