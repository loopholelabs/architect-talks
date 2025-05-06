# Drafter k3s Migration

```shell
export HOSTS=(
  # AWS Servers - us_east_2
  "ec2-user@3.149.235.239"

  # Azure Servers - Central US
  "azure-user@52.176.104.214"

  # GCP Servers - us_east4_b
  "gcp-user@34.86.104.210"

  # Hetzner Servers - HIL
  "root@5.78.65.251"
)

parallel --ungroup --jobs 1 'ssh -o StrictHostKeychecking=no -tt {} "uname -r && cat /proc/cpuinfo | grep \"model name\|address sizes\" | sort | uniq && sudo dmidecode -t processor | grep \"Max Speed\" | sort | uniq && systemd-detect-virt && ls /sys/module/ | grep kvm"' ::: "${HOSTS[@]}"

ssh -tt ${HOSTS[0]} "sudo drafter-nat --host-interface eth0"
ssh -tt ${HOSTS[1]} "sudo drafter-nat --host-interface eth0"
ssh -tt ${HOSTS[2]} "sudo drafter-nat --host-interface eth0"
ssh -tt ${HOSTS[3]} "sudo drafter-nat --host-interface eth0"

# For AWS, we need to use the internal IP since traffic from the public IP gets it's IP rewritten to the internal one before arriving at the NAT
ssh -tt "${HOSTS[0]}" "sudo drafter-forwarder --port-forwards '[
  {
    \"netns\": \"ark0\",
    \"internalPort\": \"6443\",
    \"protocol\": \"tcp\",
    \"externalAddr\": \"127.0.0.1:6443\"
  },
  {
    \"netns\": \"ark0\",
    \"internalPort\": \"6443\",
    \"protocol\": \"tcp\",
    \"externalAddr\": \"10.0.1.112:6443\"
  }
]'"
# For Azure, we need to use the internal IP since traffic from the public IP gets it's IP rewritten to the internal one before arriving at the NAT
ssh -tt "${HOSTS[1]}" "sudo drafter-forwarder --port-forwards '[
  {
    \"netns\": \"ark0\",
    \"internalPort\": \"6379\",
    \"protocol\": \"tcp\",
    \"externalAddr\": \"127.0.0.1:3333\"
  },
  {
    \"netns\": \"ark0\",
    \"internalPort\": \"6379\",
    \"protocol\": \"tcp\",
    \"externalAddr\": \"10.0.1.4:3333\"
  }
]'"
# For GCP, we need to use the internal IP since traffic from the public IP gets it's IP rewritten to the internal one before arriving at the NAT
ssh -tt "${HOSTS[2]}" "sudo drafter-forwarder --port-forwards '[
  {
    \"netns\": \"ark1\",
    \"internalPort\": \"6379\",
    \"protocol\": \"tcp\",
    \"externalAddr\": \"127.0.0.1:3333\"
  },
  {
    \"netns\": \"ark1\",
    \"internalPort\": \"6379\",
    \"protocol\": \"tcp\",
    \"externalAddr\": \"10.2.0.2:3333\"
  }
]'"
ssh -tt "${HOSTS[3]}" "sudo drafter-forwarder --port-forwards '[
  {
    \"netns\": \"ark1\",
    \"internalPort\": \"6379\",
    \"protocol\": \"tcp\",
    \"externalAddr\": \"127.0.0.1:3333\"
  },
  {
    \"netns\": \"ark1\",
    \"internalPort\": \"6379\",
    \"protocol\": \"tcp\",
    \"externalAddr\": \"${HOSTS[3]#*@}:3333\"
  }
]'"

# make depend/os OS_DEFCONFIG=drafteros-k3s-server-firecracker-x86_64_pvm_defconfig

# Configure external IP here
mv ~/Downloads/buildroot.server/ out/buildroot; make config/os && make build/os && mv out/buildroot/ ~/Downloads/buildroot.server && parallel --ungroup --jobs 0 'rsync -a --delete --progress --compress --compress-choice=zstd --mkpath "out/blueprint/" {}:out/blueprint/k3s-server' ::: "${HOSTS[0]}"

# make depend/os OS_DEFCONFIG=drafteros-k3s-client-firecracker-x86_64_pvm_defconfig
# Configure external IPs here
mv ~/Downloads/buildroot.client/ out/buildroot; make config/os && make build/os && mv out/buildroot/ ~/Downloads/buildroot.client && parallel --ungroup --jobs 0 'rsync -a --delete --progress --compress --compress-choice=zstd --mkpath "out/blueprint/" {}:out/blueprint/k3s-client' ::: "${HOSTS[1]}"

# Create snapshot for server & start it on first cloud host (AWS)
# Use the T2CL CPU template Intel and on T2A on AMD (at least AMD Milan)
ssh -tt ${HOSTS[0]} "sudo rmmod kvm_pvm && sudo modprobe kvm_pvm && sudo drafter-snapshotter --cpu-template T2A --cpu-count 4 -memory-size 2048 --netns ark0 --devices '[
  {
    \"name\": \"state\",
    \"output\": \"out/package/k3s-server/state.bin\"
  },
  {
    \"name\": \"memory\",
    \"output\": \"out/package/k3s-server/memory.bin\"
  },
  {
    \"name\": \"kernel\",
    \"input\": \"out/blueprint/k3s-server/vmlinux\",
    \"output\": \"out/package/k3s-server/vmlinux\"
  },
  {
    \"name\": \"disk\",
    \"input\": \"out/blueprint/k3s-server/rootfs.ext4\",
    \"output\": \"out/package/k3s-server/rootfs.ext4\"
  },
  {
    \"name\": \"config\",
    \"output\": \"out/package/k3s-server/config.json\"
  }
]' && sudo rm -rf out/instance-0/k3s-server && sudo drafter-peer --enable-input --netns ark0 --raddr '' --laddr '0.0.0.0:1337' --devices '[
  {
    \"name\": \"state\",
    \"base\": \"out/package/k3s-server/state.bin\",
    \"overlay\": \"out/instance-0/k3s-server/overlay/state.bin\",
    \"state\": \"out/instance-0/k3s-server/state/state.bin\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"memory\",
    \"base\": \"out/package/k3s-server/memory.bin\",
    \"overlay\": \"out/instance-0/k3s-server/overlay/memory.bin\",
    \"state\": \"out/instance-0/k3s-server/state/memory.bin\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"kernel\",
    \"base\": \"out/package/k3s-server/vmlinux\",
    \"overlay\": \"out/instance-0/k3s-server/overlay/vmlinux\",
    \"state\": \"out/instance-0/k3s-server/state/vmlinux\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"disk\",
    \"base\": \"out/package/k3s-server/rootfs.ext4\",
    \"overlay\": \"out/instance-0/k3s-server/overlay/rootfs.ext4\",
    \"state\": \"out/instance-0/k3s-server/state/rootfs.ext4\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"config\",
    \"base\": \"out/package/k3s-server/config.json\",
    \"overlay\": \"out/instance-0/k3s-server/overlay/config.json\",
    \"state\": \"out/instance-0/k3s-server/state/config.json\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  }
]'"

cat /etc/rancher/k3s/k3s.yaml # Place into ~/.kube/config on your local workstation & replace https://127.0.0.1:6443 with https://3.149.235.239:6443

# Create snapshot for agent & start it on second cloud host (Azure)
# Use the T2CL CPU template Intel and on T2A on AMD (at least AMD Milan)
ssh -tt ${HOSTS[1]} "sudo rmmod kvm_pvm && sudo modprobe kvm_pvm && sudo drafter-snapshotter --cpu-template T2A --cpu-count 4 -memory-size 2048 --netns ark0 --devices '[
  {
    \"name\": \"state\",
    \"output\": \"out/package/k3s-client/state.bin\"
  },
  {
    \"name\": \"memory\",
    \"output\": \"out/package/k3s-client/memory.bin\"
  },
  {
    \"name\": \"kernel\",
    \"input\": \"out/blueprint/k3s-client/vmlinux\",
    \"output\": \"out/package/k3s-client/vmlinux\"
  },
  {
    \"name\": \"disk\",
    \"input\": \"out/blueprint/k3s-client/rootfs.ext4\",
    \"output\": \"out/package/k3s-client/rootfs.ext4\"
  },
  {
    \"name\": \"config\",
    \"output\": \"out/package/k3s-client/config.json\"
  }
]' && sudo rm -rf out/instance-0/k3s-client && sudo drafter-peer --enable-input --netns ark0 --raddr '' --laddr '0.0.0.0:1337' --devices '[
  {
    \"name\": \"state\",
    \"base\": \"out/package/k3s-client/state.bin\",
    \"overlay\": \"out/instance-0/k3s-client/overlay/state.bin\",
    \"state\": \"out/instance-0/k3s-client/state/state.bin\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"memory\",
    \"base\": \"out/package/k3s-client/memory.bin\",
    \"overlay\": \"out/instance-0/k3s-client/overlay/memory.bin\",
    \"state\": \"out/instance-0/k3s-client/state/memory.bin\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"kernel\",
    \"base\": \"out/package/k3s-client/vmlinux\",
    \"overlay\": \"out/instance-0/k3s-client/overlay/vmlinux\",
    \"state\": \"out/instance-0/k3s-client/state/vmlinux\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"disk\",
    \"base\": \"out/package/k3s-client/rootfs.ext4\",
    \"overlay\": \"out/instance-0/k3s-client/overlay/rootfs.ext4\",
    \"state\": \"out/instance-0/k3s-client/state/rootfs.ext4\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"config\",
    \"base\": \"out/package/k3s-client/config.json\",
    \"overlay\": \"out/instance-0/k3s-client/overlay/config.json\",
    \"state\": \"out/instance-0/k3s-client/state/config.json\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  }
]'"

kubectl apply -f "/home/pojntfx/Documents/Talks/KubeCon NA 2024/valkey_k8s_pod_k3s.yaml" # Be sure to adjust the node name first

# Migrate snapshot to third cloud host (GCP)
ssh -tt ${HOSTS[2]} "sudo rmmod kvm_pvm && sudo modprobe kvm_pvm && sudo rm -rf out/instance-1/k3s-client && sudo drafter-peer --netns ark1 --raddr '${HOSTS[1]#*@}:1337' --laddr '0.0.0.0:1337' --devices '[
  {
    \"name\": \"state\",
    \"base\": \"out/package/k3s-client/state.bin\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/state.bin\",
    \"state\": \"out/instance-1/k3s-client/state/state.bin\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"memory\",
    \"base\": \"out/package/k3s-client/memory.bin\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/memory.bin\",
    \"state\": \"out/instance-1/k3s-client/state/memory.bin\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"kernel\",
    \"base\": \"out/package/k3s-client/vmlinux\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/vmlinux\",
    \"state\": \"out/instance-1/k3s-client/state/vmlinux\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"disk\",
    \"base\": \"out/package/k3s-client/rootfs.ext4\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/rootfs.ext4\",
    \"state\": \"out/instance-1/k3s-client/state/rootfs.ext4\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"config\",
    \"base\": \"out/package/k3s-client/config.json\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/config.json\",
    \"state\": \"out/instance-1/k3s-client/state/config.json\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  }
]'"

# # Migrate snapshot to fourth cloud host (Hetzner)
# ssh -tt ${HOSTS[3]} "sudo rmmod kvm_pvm && sudo modprobe kvm_pvm && sudo rm -rf out/instance-1/k3s-client && sudo drafter-peer --netns ark1 --raddr '${HOSTS[2]#*@}:1337' --laddr '0.0.0.0:1337' --devices '[
#   {
#     \"name\": \"state\",
#     \"base\": \"out/package/k3s-client/state.bin\",
#     \"overlay\": \"out/instance-1/k3s-client/overlay/state.bin\",
#     \"state\": \"out/instance-1/k3s-client/state/state.bin\",
#     \"blockSize\": 65536,
#     \"expiry\": 1000000000,
#     \"maxDirtyBlocks\": 200,
#     \"minCycles\": 5,
#     \"maxCycles\": 20,
#     \"cycleThrottle\": 500000000,
#     \"makeMigratable\": true,
#     \"shared\": false
#   },
#   {
#     \"name\": \"memory\",
#     \"base\": \"out/package/k3s-client/memory.bin\",
#     \"overlay\": \"out/instance-1/k3s-client/overlay/memory.bin\",
#     \"state\": \"out/instance-1/k3s-client/state/memory.bin\",
#     \"blockSize\": 65536,
#     \"expiry\": 1000000000,
#     \"maxDirtyBlocks\": 200,
#     \"minCycles\": 5,
#     \"maxCycles\": 20,
#     \"cycleThrottle\": 500000000,
#     \"makeMigratable\": true,
#     \"shared\": false
#   },
#   {
#     \"name\": \"kernel\",
#     \"base\": \"out/package/k3s-client/vmlinux\",
#     \"overlay\": \"out/instance-1/k3s-client/overlay/vmlinux\",
#     \"state\": \"out/instance-1/k3s-client/state/vmlinux\",
#     \"blockSize\": 65536,
#     \"expiry\": 1000000000,
#     \"maxDirtyBlocks\": 200,
#     \"minCycles\": 5,
#     \"maxCycles\": 20,
#     \"cycleThrottle\": 500000000,
#     \"makeMigratable\": true,
#     \"shared\": false
#   },
#   {
#     \"name\": \"disk\",
#     \"base\": \"out/package/k3s-client/rootfs.ext4\",
#     \"overlay\": \"out/instance-1/k3s-client/overlay/rootfs.ext4\",
#     \"state\": \"out/instance-1/k3s-client/state/rootfs.ext4\",
#     \"blockSize\": 65536,
#     \"expiry\": 1000000000,
#     \"maxDirtyBlocks\": 200,
#     \"minCycles\": 5,
#     \"maxCycles\": 20,
#     \"cycleThrottle\": 500000000,
#     \"makeMigratable\": true,
#     \"shared\": false
#   },
#   {
#     \"name\": \"config\",
#     \"base\": \"out/package/k3s-client/config.json\",
#     \"overlay\": \"out/instance-1/k3s-client/overlay/config.json\",
#     \"state\": \"out/instance-1/k3s-client/state/config.json\",
#     \"blockSize\": 65536,
#     \"expiry\": 1000000000,
#     \"maxDirtyBlocks\": 200,
#     \"minCycles\": 5,
#     \"maxCycles\": 20,
#     \"cycleThrottle\": 500000000,
#     \"makeMigratable\": true,
#     \"shared\": false
#   }
# ]'"

# Migrate snapshot back to second cloud host (Azure)
ssh -tt ${HOSTS[1]} "sudo rmmod kvm_pvm && sudo modprobe kvm_pvm && sudo rm -rf out/instance-1/k3s-client && sudo drafter-peer --netns ark0 --raddr '${HOSTS[2]#*@}:1337' --laddr '0.0.0.0:1337' --devices '[
  {
    \"name\": \"state\",
    \"base\": \"out/package/k3s-client/state.bin\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/state.bin\",
    \"state\": \"out/instance-1/k3s-client/state/state.bin\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"memory\",
    \"base\": \"out/package/k3s-client/memory.bin\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/memory.bin\",
    \"state\": \"out/instance-1/k3s-client/state/memory.bin\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"kernel\",
    \"base\": \"out/package/k3s-client/vmlinux\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/vmlinux\",
    \"state\": \"out/instance-1/k3s-client/state/vmlinux\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"disk\",
    \"base\": \"out/package/k3s-client/rootfs.ext4\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/rootfs.ext4\",
    \"state\": \"out/instance-1/k3s-client/state/rootfs.ext4\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  },
  {
    \"name\": \"config\",
    \"base\": \"out/package/k3s-client/config.json\",
    \"overlay\": \"out/instance-1/k3s-client/overlay/config.json\",
    \"state\": \"out/instance-1/k3s-client/state/config.json\",
    \"blockSize\": 65536,
    \"expiry\": 1000000000,
    \"maxDirtyBlocks\": 200,
    \"minCycles\": 5,
    \"maxCycles\": 20,
    \"cycleThrottle\": 500000000,
    \"makeMigratable\": true,
    \"shared\": false
  }
]'"
```
