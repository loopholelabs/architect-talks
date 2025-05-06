# Architect k3s Migration

```shell
export HOSTS=(
  # AWS Servers - us_west_2
  "ec2-user@35.166.102.13"

  # Azure Servers - West US
  "azure-user@13.87.204.119"

  # GCP Servers - us_west1_a
  "gcp-user@34.168.116.120"

  # AWS Servers - us_west_2
  "ec2-user@44.232.34.249"

  # Azure Servers - West US
  "azure-user@13.88.61.183"

  # AWS Servers - us_west_2
  "ec2-user@35.93.209.14"

  # Hetzner Servers - HIL
  "root@5.78.94.96"
)

parallel --ungroup --jobs 1 'ssh -o StrictHostKeychecking=no -tt {} "uname -r && cat /proc/cpuinfo | grep \"model name\|address sizes\" | sort | uniq && sudo dmidecode -t processor | grep \"Max Speed\" | sort | uniq && systemd-detect-virt && ls /sys/module/ | grep kvm"' ::: "${HOSTS[@]}"

parallel --ungroup --jobs 1 'ssh -o StrictHostKeychecking=no -tt {} "sudo pkill -2 architect"' ::: "${HOSTS[@]}"

# make depend/os OS_DEFCONFIG=architectos-k3s-server-firecracker-x86_64_pvm_defconfig
# make depend/os OS_DEFCONFIG=architectos-k3s-client-firecracker-x86_64_pvm_defconfig

# Configure external IPs here - use 35.166.102.13 as the IP for both server and agent
sudo pkill -2 architect; mv ~/Downloads/buildroot-architect.server/ out/buildroot; make config/os && make build/os && sudo mkdir -p /root/.local/share/architect/blueprints/k3s-server && sudo cp out/blueprint/* /root/.local/share/architect/blueprints/k3s-server && mv out/buildroot/ ~/Downloads/buildroot-architect.server && sudo pkill -2 architect; mv ~/Downloads/buildroot-architect.client/ out/buildroot; make config/os && make build/os && sudo mkdir -p /root/.local/share/architect/blueprints/k3s-client && sudo cp out/blueprint/* /root/.local/share/architect/blueprints/k3s-client && mv out/buildroot/ ~/Downloads/buildroot-architect.client && sudo -E parallel --ungroup --jobs 0 'rsync -a --delete --progress --compress --compress-choice=zstd --mkpath -e "ssh -i /home/pojntfx/.ssh/id_rsa -o StrictHostKeychecking=no" "/root/.local/share/architect/blueprints/" {}:architect-blueprints/ && ssh {} "sudo mkdir -p /root/.local/share/architect/blueprints && sudo cp -r architect-blueprints/. /root/.local/share/architect/blueprints/"' ::: "${HOSTS[@]}"
```

```shell
# On Ingress
ssh "${HOSTS[0]}"

sudo mkdir -p /root/.config
sudo vi /root/.config/architect.yaml
# ingress:
#   httpAddr: /ip4/0.0.0.0/tcp/3030
#   hostInterface: eth0

echo 'bind 0.0.0.0' | sudo tee -a /etc/redis/redis.conf

sudo dnf install -y redis
sudo systemctl enable --now redis
# nc 35.166.102.13 6379 del mytable

sudo ip link set dev eth0 mtu 1460

# On second cloud host (Azure)
ssh "${HOSTS[1]}"

sudo mkdir -p /root/.config
sudo vi /root/.config/architect.yaml
# addr: /ip4/0.0.0.0/tcp/1337
# workerAddr: /ip4/0.0.0.0/tcp/1337
# workerConfig:
#   migrationListenIP: 10.2.0.2
#   cri:
#     valkeyURL: redis://35.166.102.13:6379/0
#   telemetry:
#     metrics:
#       addr: ''
#   networking:
#     migrationStateStoreURL: redis://35.166.102.13:6379/0
#     hostInterface: eth0
#     useLocalRouting: false
#     ingressControllerAddr: /ip4/35.166.102.13/tcp/3030
#     ingressControllerTargetIP: 13.87.204.119
#     ingressControllerDestinationIP: 10.0.1.224 # This IP needs to be the private IP of the ingress, or the public IP, whichever one is attached to the ingress' interface

sudo ip link set dev eth0 mtu 1460

# On third cloud host (GCP)
ssh "${HOSTS[2]}"

sudo mkdir -p /root/.config
sudo vi /root/.config/architect.yaml
# addr: /ip4/0.0.0.0/tcp/1337
# workerAddr: /ip4/0.0.0.0/tcp/1337
# workerConfig:
#   migrationListenIP: 10.2.0.2
#   cri:
#     valkeyURL: redis://35.166.102.13:6379/0
#   telemetry:
#     metrics:
#       addr: ''
#   networking:
#     migrationStateStoreURL: redis://35.166.102.13:6379/0
#     hostInterface: eth0
#     useLocalRouting: true
#     ingressControllerAddr: /ip4/35.166.102.13/tcp/3030
#     ingressControllerTargetIP: 34.168.116.120
#     ingressControllerDestinationIP: 10.0.1.224 # This IP needs to be the private IP of the ingress, or the public IP, whichever one is attached to the ingress' interface

sudo ip link set dev eth0 mtu 1460

# On fourth cloud host (AWS)
ssh "${HOSTS[3]}"

sudo mkdir -p /root/.config
sudo vi /root/.config/architect.yaml
# addr: /ip4/0.0.0.0/tcp/1337
# workerAddr: /ip4/0.0.0.0/tcp/1337
# workerConfig:
#   migrationListenIP: 10.0.1.58
#   cri:
#     valkeyURL: redis://172.31.52.200:6380/0
#   telemetry:
#     metrics:
#       addr: ''
#   networking:
#     migrationStateStoreURL: redis://35.166.102.13:6379/0
#     hostInterface: eth0
#     useLocalRouting: true
#     ingressControllerAddr: /ip4/35.166.102.13/tcp/3030
#     ingressControllerTargetIP: 44.232.34.249
#     ingressControllerDestinationIP: 10.0.1.224 # This IP needs to be the private IP of the ingress, or the public IP, whichever one is attached to the ingress' interface

sudo ip link set dev eth0 mtu 1460

# On fifth cloud host (Azure)
ssh "${HOSTS[4]}"

sudo mkdir -p /root/.config
sudo vi /root/.config/architect.yaml
# addr: /ip4/0.0.0.0/tcp/1337
# workerAddr: /ip4/0.0.0.0/tcp/1337
# workerConfig:
#   migrationListenIP: 10.0.1.4
#   cri:
#     valkeyURL: redis://172.31.52.200:6380/0
#   telemetry:
#     metrics:
#       addr: ''
#   networking:
#     migrationStateStoreURL: redis://35.166.102.13:6379/0
#     hostInterface: eth0
#     useLocalRouting: false
#     ingressControllerAddr: /ip4/35.166.102.13/tcp/3030
#     ingressControllerTargetIP: 13.88.61.183
#     ingressControllerDestinationIP: 10.0.1.224 # This IP needs to be the private IP of the ingress, or the public IP, whichever one is attached to the ingress' interface

sudo ip link set dev eth0 mtu 1460

# On sixth cloud host (AWS)
ssh "${HOSTS[5]}"

sudo mkdir -p /root/.config
sudo vi /root/.config/architect.yaml
# addr: /ip4/0.0.0.0/tcp/1337
# workerAddr: /ip4/0.0.0.0/tcp/1337
# workerConfig:
#   migrationListenIP: 10.0.1.77
#   cri:
#     valkeyURL: redis://172.31.52.200:6380/0
#   telemetry:
#     metrics:
#       addr: ''
#   networking:
#     migrationStateStoreURL: redis://35.166.102.13:6379/0
#     hostInterface: eth0
#     useLocalRouting: true
#     ingressControllerAddr: /ip4/35.166.102.13/tcp/3030
#     ingressControllerTargetIP: 35.93.209.14
#     ingressControllerDestinationIP: 10.0.1.224 # This IP needs to be the private IP of the ingress, or the public IP, whichever one is attached to the ingress' interface

sudo ip link set dev eth0 mtu 1460
```

```shell
parallel --ungroup --jobs 1 'ssh -o StrictHostKeychecking=no -tt {} "sudo pkill -2 architect"' ::: "${HOSTS[@]}"

# Start ingress on first cloud host (AWS)
CGO_ENABLED=0 GOFLAGS="-tags=exclude_graphdriver_devicemapper,exclude_graphdriver_btrfs,containers_image_openpgp" make build/architect && rsync -a --delete --progress --compress --compress-choice=zstd "out/architect" ${HOSTS[0]}:/tmp/architect && ssh -tt "${HOSTS[0]}" sudo /tmp/architect ingress --reserved-ports-start 11000

# Start worker on second cloud host (Azure)
CGO_ENABLED=0 GOFLAGS="-tags=exclude_graphdriver_devicemapper,exclude_graphdriver_btrfs,containers_image_openpgp" make build/architect && rsync -a --delete --progress --compress --compress-choice=zstd "out/architect" ${HOSTS[1]}:/tmp/architect && ssh -tt "${HOSTS[1]}" sudo /tmp/architect worker --enable-input

# Start worker on third cloud host (GCP)
CGO_ENABLED=0 GOFLAGS="-tags=exclude_graphdriver_devicemapper,exclude_graphdriver_btrfs,containers_image_openpgp" make build/architect && rsync -a --delete --progress --compress --compress-choice=zstd "out/architect" ${HOSTS[2]}:/tmp/architect && ssh -tt "${HOSTS[2]}" sudo /tmp/architect worker --enable-input

# Start worker on fourth cloud host (AWS)
CGO_ENABLED=0 GOFLAGS="-tags=exclude_graphdriver_devicemapper,exclude_graphdriver_btrfs,containers_image_openpgp" make build/architect && rsync -a --delete --progress --compress --compress-choice=zstd "out/architect" ${HOSTS[3]}:/tmp/architect && ssh -tt "${HOSTS[3]}" sudo /tmp/architect worker --enable-input

# Start worker on fifth cloud host (Azure)
CGO_ENABLED=0 GOFLAGS="-tags=exclude_graphdriver_devicemapper,exclude_graphdriver_btrfs,containers_image_openpgp" make build/architect && rsync -a --delete --progress --compress --compress-choice=zstd "out/architect" ${HOSTS[4]}:/tmp/architect && ssh -tt "${HOSTS[4]}" sudo /tmp/architect worker --enable-input

# Start worker on sixth cloud host (AWS)
CGO_ENABLED=0 GOFLAGS="-tags=exclude_graphdriver_devicemapper,exclude_graphdriver_btrfs,containers_image_openpgp" make build/architect && rsync -a --delete --progress --compress --compress-choice=zstd "out/architect" ${HOSTS[5]}:/tmp/architect && ssh -tt "${HOSTS[5]}" sudo /tmp/architect worker --enable-input
```

```shell
# On local workstation
vi ~/.config/arc.yaml
workerAddr: /ip4/13.87.204.119/tcp/1337
# # workerAddr: /ip4/34.168.116.120/tcp/1337
# # workerAddr: /ip4/44.232.34.249/tcp/1337
# # workerAddr: /ip4/13.88.61.183/tcp/1337
# # workerAddr: /ip4/35.93.209.14/tcp/1337
# ingressControllerAddr: /ip4/35.166.102.13/tcp/3030

# Create snapshot for server on second cloud host (Azure)
make build/arc && sudo make install/arc && (arc -f json instance list | jq -r '.[] | select(.running) | "arc instance stop \(.name)"'; arc -f json instance list | jq -r '.[] | "arc instance delete \(.name)"'; arc -f json package list | jq -r '.[] | "arc package delete \(.name)"') | sh; arc package create --template T2A --cpus 4 --memory 2048 k3s-server k3s-server

# Start snapshot on second cloud host (Azure)
arc instance create architect-k3s-server-1 k3s-server --skip-oci-device --publish 6443:6443

cat /etc/rancher/k3s/k3s.yaml # Place into ~/.kube/config on your local workstation & replace https://127.0.0.1:6443 with https://35.166.102.13:6443

arc -f json instance list
arc -f json ingress listener list
arc -f json instance port list architect-k3s-server-1

# On local workstation
vi ~/.config/arc.yaml

# Create snapshot for agent on third cloud host (GCP)
make build/arc && sudo make install/arc && (arc -f json instance list | jq -r '.[] | select(.running) | "arc instance stop \(.name)"'; arc -f json instance list | jq -r '.[] | "arc instance delete \(.name)"'; arc -f json package list | jq -r '.[] | "arc package delete \(.name)"') | sh; arc package create --template T2A --cpus 4 --memory 2048 k3s-client k3s-client

# Start snapshot on third cloud host (GCP)
arc instance create architect-k3s-client-1 k3s-client --skip-oci-device --publish 3333:6379

kubectl apply -f "/home/pojntfx/Documents/Talks/KubeCon NA 2024/valkey_k8s_pod_k3s.yaml" # Be sure to adjust the node name first

nc 35.166.102.13 3333

# Make instance on third cloud host migratable (GCP)
arc instance set-migratable architect-k3s-client-1 true # /ip4/10.2.0.2/tcp/33807

vi ~/.config/arc.yaml # Switch to fourth host (AWS)

# Clean snapshots from fourth cloud host (AWS)
make build/arc && sudo make install/arc && (arc -f json instance list | jq -r '.[] | select(.running) | "arc instance stop \(.name)"'; arc -f json instance list | jq -r '.[] | "arc instance delete \(.name)"'; arc -f json package list | jq -r '.[] | "arc package delete \(.name)"') | sh

# Migrate instance to fourth host (AWS)
arc instance migrate-from architect-k3s-client-1 /ip4/34.168.116.120/tcp/45355 --skip-oci-device --publish 3333:6379

# Make instance on fourth cloud host migratable (AWS)
arc instance set-migratable architect-k3s-client-1 true # /ip4/10.2.0.2/tcp/33807

# On local workstation
vi ~/.config/arc.yaml # Switch to fifth host (Azure)

# Clean snapshots from fifth cloud host (Azure)
make build/arc && sudo make install/arc && (arc -f json instance list | jq -r '.[] | select(.running) | "arc instance stop \(.name)"'; arc -f json instance list | jq -r '.[] | "arc instance delete \(.name)"'; arc -f json package list | jq -r '.[] | "arc package delete \(.name)"') | sh

# Migrate instance to fifth host (Azure)
arc instance migrate-from architect-k3s-client-1 //ip444.232.34.249/tcp/43237 --skip-oci-device --publish 3333:6379

# Make instance on fifth cloud host migratable (Azure)
arc instance set-migratable architect-k3s-client-1 true # /ip4/10.2.0.2/tcp/33807

# On local workstation
vi ~/.config/arc.yaml # Switch to sixth host (AWS)

# Clean snapshots from sixth cloud host (AWS)
make build/arc && sudo make install/arc && (arc -f json instance list | jq -r '.[] | select(.running) | "arc instance stop \(.name)"'; arc -f json instance list | jq -r '.[] | "arc instance delete \(.name)"'; arc -f json package list | jq -r '.[] | "arc package delete \(.name)"') | sh

# Migrate instance to sixth host (AWS)
arc instance migrate-from architect-k3s-client-1 /ip4/13.88.61.183/tcp/36141 --skip-oci-device --publish 3333:6379
```
