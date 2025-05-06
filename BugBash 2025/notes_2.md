# Commoditize Your CI Compute: Spot Instances Without the Spotty Reliability

The story:

- Intro
  - Welcome the audience
  - Introduce myself
  - Introduce the title (but with a twist - single line, no chapters)
- Show of hands: Who here is _not_ running their CI/CD on spot instances/ephemeral compute?
- Why everybody should be using spot instances/ephemeral compute
- Why nobody is using it right now
- What would have to be done to use them
- How Architect allows you to use it
- How Architect works
- Show that Architect works with it (demo)
- How you can use Architect now (waitlist)

Drop the chapters - add the chapters to the intro we do in the beginning, to the story we tell

- Intro
  - Welcome the audience
  - Introduce myself
  - Introduce the title
  - Chapters
- Show of hands: Who here is _not_ using spot/ephemeral compute for CI/CD?
- Benefits of spot/ephemeral compute
- Why aren't we using spot/ephemeral compute for more things?
- It's not just cost: Lock-in is the actual core problem

- Tech stack that makes it possible
  - PVM: Core tech
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
  - Mirage: GPU migrations
  - Architect: Tying it all together
    - Does the "delicate dance" of orchestrating Drafter, Silo, Conduit and Mirage to live migrate from A to B with a single API call
    - Doesn't need to worr
- Examples of what Architect's policy engine can do
  - Move to a different cloud provider if it is cheaper without downtime
  - Evacuate a host if it gets the spot instance pre-emption notice
- Demo
- Conclusion
- Public beta
- Outro
  - About myself (remove the QR code)
  - About Loophole (remove the QR code)
  - QR with all links in case anyone missed it
