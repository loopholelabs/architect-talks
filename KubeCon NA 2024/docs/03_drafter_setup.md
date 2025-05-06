# Drafter Setup

```shell
parallel --ungroup --jobs 0 'ssh -tt {} "sudo dnf install -y iptables rsync && sudo systemctl disable --now firewalld; sudo modprobe nbd nbds_max=4096; sudo iptables -F; sudo iptables -X; sudo iptables -t nat -F; sudo iptables -t nat -X; sudo iptables -t mangle -F; sudo iptables -t mangle -X; sudo iptables -P INPUT ACCEPT; sudo iptables -P FORWARD ACCEPT; sudo iptables -P OUTPUT ACCEPT; sudo ip link set dev eth0 mtu 1460"' ::: "${HOSTS[@]}"

# Make sure you have a Firecracker binary _with_ PVM support selected
tools/devtool build --release && sudo install ./build/cargo_target/x86_64-unknown-linux-musl/release/{firecracker,jailer} /usr/local/bin && parallel --ungroup --jobs 0 '
  rsync -a --delete --progress --compress --compress-choice=zstd \
    --mkpath "$(which firecracker)" "$(which jailer)" \
    {}:/tmp &&
  ssh -tt {} "sudo install /tmp/{firecracker,jailer} /usr/bin"
' ::: "${HOSTS[@]}"

make -j$(nproc) && sudo make install -j$(nproc) && parallel --ungroup --jobs 0 '
  rsync -a --delete --progress --compress --compress-choice=zstd \
    --mkpath out/drafter* \
    {}:/tmp &&
  ssh -tt {} "sudo install /tmp/drafter* /usr/bin"
' ::: "${HOSTS[@]}"
```
