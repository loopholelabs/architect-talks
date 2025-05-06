We're in the advanced/emerging track, and the talk is about PVM & how we use it mostly, but I want to make sure we still direct most people to Architect in the end without sounding like too much of a shill :slight_smile:

Right now my plan is roughly this:

- Short speaker & company intro, outline
- Overview of current limitations of VPS-based services, esp. spot (no VMs due to no nested virt = no migration, CRIU is too unreliable, networking connections break)
- Value proposition of running VMs on VMs vs. using cloud services directly, market reasons why cloud providers usually don't allow nested virt usually (basically going over https://gwern.net/complement)
- Introduction of PVM and how it solves that issue by dropping the need for the cloud provider to consent to nested virt, how to install & evaluate it, what the upstreaming status is
- Why PVM by itself needs Loophole's patches to OSS components to be useful (our own PVM patches, patches to Firecracker, Drafter as the core technology, Silo as the transport layer)
- Short example of how we run Kubernetes inside of Drafter (basically packaging)
- Migrating a k8s cluster from EC2 to GCP to Azure with Drafter (this is quite technical since Drafter is low-level, network connections will break). Shows that the core tech isn't just smoke and mirrors and that it's reproducible
- Limitations of OSS Drafter as a core technology and why you should use Architect in production (simpler API, network connections don't break, SaaS option)
- Showing the same migration, but this time with Architect (should be much simpler since it' a single API call, and demo the network migration)
- Ending the talk with some more short (~1 minute) demos of Architect's other modes, esp. the native k8s integration and CI/CD
- Linking to Loophole sites, repos and papers used
