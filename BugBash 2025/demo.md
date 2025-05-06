# BugBash 2025 Demo

```shell
eval $(tofu -chdir=auth output --json | ./scripts/cloud-opentofu-to-auth-env.sh)
export ANSIBLE_GATHERING=explicit ANSIBLE_PIPELINING=true ANSIBLE_HOST_KEY_CHECKING=false
ansible-playbook -i inventory.json playbooks/setup_architect.yaml -e "architect_source=local architect_local_path=${PWD}/.. architect_goarch=amd64"
```

```shell
HIDE_CONDUIT_LOGS=true HIDE_RPC_LOGS=true HIDE_METRICS_LOGS=true HIDE_MIGRATION_PROGRESS=true ./scripts/cloud-stream-logs.sh | tee log
```

```shell
./scripts/cloud-stream-logs-crun.sh
```

```shell
DISABLE_POSTCOPY_MIGRATION=true ENABLE_COW=true DISABLE_CONTINOUS_MIGRATION=true ENABLE_LARGE_BLOCKS=false REDUCE_P2P_CONCURRENCY=false REDUCE_S3_CONCURRENCY=false DISABLE_PACKAGE_REBUILD=true MAX_MIGRATIONS=500 OCI_IMAGE=docker://docker.io/pojntfx/tigerbeetle-fuzzer:latest EXPOSE_PORTS=6333:6379 ./scripts/cloud-migration-loop.sh
```

https://grafana.dev.architect.run/d/aedrct2pach6ob/migrations?kiosk=&orgId=1&from=now-30s&to=now&timezone=utc&var-src=pojntfx-alma-gcp-1-us-west1-a&var-dest=pojntfx-alma-gcp-2-us-west1-a&var-device=config

![](https://media.discordapp.net/attachments/1113823188661571624/1357310787407970364/image.png?ex=67f10f2d&is=67efbdad&hm=c98733bf5108289b36f4e4679147cbf87a917d42291e3767cea03182edf2f8f4&=&format=webp&quality=lossless&width=2326&height=1416)
./scripts/cloud-stream-logs-crun.sh
