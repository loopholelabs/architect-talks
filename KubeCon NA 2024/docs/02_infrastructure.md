# Uninstance OpenTofu Development

```shell
export TF_VAR_hetzner_api_key=asdf TF_VAR_aws_access_key=asdf TF_VAR_aws_secret_key=asdf TF_VAR_aws_token=asdf TF_VAR_azure_subscription_id=asdf TF_VAR_equinix_project_id=asdf TF_VAR_equinix_auth_token=asdf TF_VAR_gcp_project_id=asdf

tofu plan

tofu apply --auto-approve

# aws_servers = {
#   "us_east_2" = {
#     "alma_aws_pvm_node_1_us_east_2" = {
#       "ipv4_address" = "3.141.164.143"
#       "ipv6_address" = "2600:1f16:19c:ce01:7b36:ef7:50f4:4366"
#       "user" = "ec2-user"
#     }
#     "alma_aws_pvm_node_2_us_east_2" = {
#       "ipv4_address" = "3.149.235.239"
#       "ipv6_address" = "2600:1f16:143c:3501:6742:fc51:e8c0:8715"
#       "user" = "ec2-user"
#     }
#     "alma_aws_pvm_node_3_us_east_2" = {
#       "ipv4_address" = "3.144.90.2"
#       "ipv6_address" = "2600:1f16:18f0:4101:3f3b:648c:d81e:bd26"
#       "user" = "ec2-user"
#     }
#   }
#   "us_west_2" = {
#     "alma_aws_pvm_node_1_us_west_2" = {
#       "ipv4_address" = "35.93.209.14"
#       "ipv6_address" = "2600:1f14:a04:5e01:3e87:a720:1e40:47b6"
#       "user" = "ec2-user"
#     }
#     "alma_aws_pvm_node_2_us_west_2" = {
#       "ipv4_address" = "35.166.102.13"
#       "ipv6_address" = "2600:1f14:1a62:201:9e7d:e586:92b:6840"
#       "user" = "ec2-user"
#     }
#     "alma_aws_pvm_node_3_us_west_2" = {
#       "ipv4_address" = "44.232.34.249"
#       "ipv6_address" = "2600:1f14:1eb:8001:3f73:a44c:29e0:1d87"
#       "user" = "ec2-user"
#     }
#   }
# }
# azure_servers = {
#   "global" = {
#     "alma_azure_pvm_node_1_central_us" = {
#       "ipv4_address" = "104.208.29.229"
#       "ipv6_address" = "2603:1030:7:5::106"
#       "user" = "azure-user"
#     }
#     "alma_azure_pvm_node_1_west_us" = {
#       "ipv4_address" = "13.88.61.183"
#       "ipv6_address" = "2603:1030:a02:9::33"
#       "user" = "azure-user"
#     }
#     "alma_azure_pvm_node_2_central_us" = {
#       "ipv4_address" = "52.176.104.214"
#       "ipv6_address" = "2603:1030:7:5::104"
#       "user" = "azure-user"
#     }
#     "alma_azure_pvm_node_2_west_us" = {
#       "ipv4_address" = "13.87.204.119"
#       "ipv6_address" = "2603:1030:a02:9::5d"
#       "user" = "azure-user"
#     }
#     "alma_azure_pvm_node_3_central_us" = {
#       "ipv4_address" = "52.165.238.135"
#       "ipv6_address" = "2603:1030:7:5::246"
#       "user" = "azure-user"
#     }
#     "alma_azure_pvm_node_3_west_us" = {
#       "ipv4_address" = "104.210.35.126"
#       "ipv6_address" = "2a01:111:f100:3002::8987:360c"
#       "user" = "azure-user"
#     }
#   }
# }
# gcp_servers = {
#   "global" = {
#     "alma_gcp_pvm_node_1_us_west1_a" = {
#       "ipv4_address" = "34.168.116.120"
#       "ipv6_address" = "2600:1900:4041:a6d:0:0:0:0"
#       "user" = "gcp-user"
#     }
#     "alma_gcp_pvm_node_2_us_west1_a" = {
#       "ipv4_address" = "35.233.246.83"
#       "ipv6_address" = "2600:1900:4041:353:0:0:0:0"
#       "user" = "gcp-user"
#     }
#     "alma_gcp_pvm_node_3_us_east4_b" = {
#       "ipv4_address" = "34.145.255.221"
#       "ipv6_address" = "2600:1900:4090:4c83:0:0:0:0"
#       "user" = "gcp-user"
#     }
#     "alma_gcp_pvm_node_4_us_east4_b" = {
#       "ipv4_address" = "34.86.104.210"
#       "ipv6_address" = "2600:1900:4090:440:0:0:0:0"
#       "user" = "gcp-user"
#     }
#     "alma_gcp_pvm_node_5_us_ctl1_b" = {
#       "ipv4_address" = "35.232.156.126"
#       "ipv6_address" = "2600:1900:4001:1ac:0:0:0:0"
#       "user" = "gcp-user"
#     }
#     "alma_gcp_pvm_node_6_us_ctl1_b" = {
#       "ipv4_address" = "34.70.162.233"
#       "ipv6_address" = "2600:1900:4000:10ea:0:0:0:0"
#       "user" = "gcp-user"
#     }
#   }
# }
# hetzner_servers = {
#   "global" = {
#     "alma_hetzner_pvm_node_1_hil" = {
#       "ipv4_address" = "5.78.120.0"
#       "ipv6_address" = "2a01:4ff:1f0:c6b2::1"
#       "user" = "root"
#     }
#     "alma_hetzner_pvm_node_2_hil" = {
#       "ipv4_address" = "5.78.65.251"
#       "ipv6_address" = "2a01:4ff:1f0:ca04::1"
#       "user" = "root"
#     }
#     "alma_hetzner_pvm_node_3_hil" = {
#       "ipv4_address" = "5.78.94.96"
#       "ipv6_address" = "2a01:4ff:1f0:d0d5::1"
#       "user" = "root"
#     }
#     "alma_hetzner_pvm_node_4_sin" = {
#       "ipv4_address" = "5.223.47.64"
#       "ipv6_address" = "2a01:4ff:2f0:36a1::1"
#       "user" = "root"
#     }
#     "alma_hetzner_pvm_node_5_sin" = {
#       "ipv4_address" = "5.223.43.225"
#       "ipv6_address" = "2a01:4ff:2f0:3669::1"
#       "user" = "root"
#     }
#     "alma_hetzner_pvm_node_6_sin" = {
#       "ipv4_address" = "5.223.50.71"
#       "ipv6_address" = "2a01:4ff:2f0:3907::1"
#       "user" = "root"
#     }
#   }
# }

# export HOSTS=("pojntfx@192.168.124.220" "pojntfx@192.168.124.121" "pojntfx@192.168.124.27")
export HOSTS=(
  # AWS Servers - us_east_2
  "ec2-user@3.141.164.143"
  "ec2-user@3.149.235.239"
  "ec2-user@3.144.90.2"

  # AWS Servers - us_west_2
  "ec2-user@35.93.209.14"
  "ec2-user@35.166.102.13"
  "ec2-user@44.232.34.249"

  # Azure Servers - Central US
  "azure-user@104.208.29.229"
  "azure-user@52.176.104.214"
  "azure-user@52.165.238.135"

  # Azure Servers - West US
  "azure-user@13.88.61.183"
  "azure-user@13.87.204.119"
  "azure-user@104.210.35.126"

  # GCP Servers - us_west1_a
  "gcp-user@34.168.116.120"
  "gcp-user@35.233.246.83"

  # GCP Servers - us_east4_b
  "gcp-user@34.145.255.221"
  "gcp-user@34.86.104.210"

  # GCP Servers - us_ctl1_b
  "gcp-user@35.232.156.126"
  "gcp-user@34.70.162.233"

  # Hetzner Servers - HIL
  "root@5.78.120.0"
  "root@5.78.65.251"
  "root@5.78.94.96"

  # Hetzner Servers - SIN
  "root@5.223.47.64"
  "root@5.223.43.225"
  "root@5.223.50.71"
)

parallel --ungroup --jobs 1 'ssh -o StrictHostKeychecking=no -tt {} "uname -r && cat /proc/cpuinfo | grep \"model name\|address sizes\" | sort | uniq && sudo dmidecode -t processor | grep \"Max Speed\" | sort | uniq && systemd-detect-virt && ls /sys/module/ | grep kvm"' ::: "${HOSTS[@]}"

# 6.7.12-pvm-host-alma-aws
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7R13 Processor
#         Max Speed: 3725 MHz
# Connection to 3.141.164.143 closed.
# 6.7.12-pvm-host-alma-aws
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7R13 Processor
#         Max Speed: 3725 MHz
# Connection to 3.149.235.239 closed.
# 6.7.12-pvm-host-alma-aws
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7R13 Processor
#         Max Speed: 3725 MHz
# Connection to 3.144.90.2 closed.
# 6.7.12-pvm-host-alma-aws
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7R13 Processor
#         Max Speed: 3725 MHz
# Connection to 35.93.209.14 closed.
# 6.7.12-pvm-host-alma-aws
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7R13 Processor
#         Max Speed: 3725 MHz
# Connection to 35.166.102.13 closed.
# 6.7.12-pvm-host-alma-aws
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7R13 Processor
#         Max Speed: 3725 MHz
# Connection to 44.232.34.249 closed.
# 6.7.12-pvm-host-alma-azure
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7763 64-Core Processor
#         Max Speed: 3525 MHz
# Connection to 104.208.29.229 closed.
# 6.7.12-pvm-host-alma-azure
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7763 64-Core Processor
#         Max Speed: 3525 MHz
# Connection to 52.176.104.214 closed.
# 6.7.12-pvm-host-alma-azure
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7763 64-Core Processor
#         Max Speed: 3525 MHz
# Connection to 52.165.238.135 closed.
# 6.7.12-pvm-host-alma-azure
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7763 64-Core Processor
#         Max Speed: 3525 MHz
# Connection to 13.88.61.183 closed.
# 6.7.12-pvm-host-alma-azure
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7763 64-Core Processor
#         Max Speed: 3525 MHz
# Connection to 13.87.204.119 closed.
# 6.7.12-pvm-host-alma-azure
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7763 64-Core Processor
#         Max Speed: 3525 MHz
# Connection to 104.210.35.126 closed.
# 6.7.12-pvm-host-alma-gcp
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7B13
#         Max Speed: 2000 MHz
# Connection to 34.168.116.120 closed.
# 6.7.12-pvm-host-alma-gcp
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7B13
#         Max Speed: 2000 MHz
# Connection to 35.233.246.83 closed.
# 6.7.12-pvm-host-alma-gcp
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7B13
#         Max Speed: 2000 MHz
# Connection to 34.145.255.221 closed.
# 6.7.12-pvm-host-alma-gcp
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7B13
#         Max Speed: 2000 MHz
# Connection to 34.86.104.210 closed.
# 6.7.12-pvm-host-alma-gcp
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7B13
#         Max Speed: 2000 MHz
# Connection to 35.232.156.126 closed.
# 6.7.12-pvm-host-alma-gcp
# address sizes   : 48 bits physical, 48 bits virtual
# model name      : AMD EPYC 7B13
#         Max Speed: 2000 MHz
# Connection to 34.70.162.233 closed.
# 6.7.12-pvm-host-alma-hetzner
# address sizes   : 40 bits physical, 48 bits virtual
# model name      : AMD EPYC Processor
#         Max Speed: 2000 MHz
# Connection to 5.78.120.0 closed.
# 6.7.12-pvm-host-alma-hetzner
# address sizes   : 40 bits physical, 48 bits virtual
# model name      : AMD EPYC Processor
#         Max Speed: 2000 MHz
# Connection to 5.78.65.251 closed.
# 6.7.12-pvm-host-alma-hetzner
# address sizes   : 40 bits physical, 48 bits virtual
# model name      : AMD EPYC Processor
#         Max Speed: 2000 MHz
# Connection to 5.78.94.96 closed.
# 6.7.12-pvm-host-alma-hetzner
# address sizes   : 40 bits physical, 48 bits virtual
# model name      : AMD EPYC Processor
#         Max Speed: 2000 MHz
# Connection to 5.223.47.64 closed.
# 6.7.12-pvm-host-alma-hetzner
# address sizes   : 40 bits physical, 48 bits virtual
# model name      : AMD EPYC Processor
#         Max Speed: 2000 MHz
# Connection to 5.223.43.225 closed.
# 6.7.12-pvm-host-alma-hetzner
# address sizes   : 40 bits physical, 48 bits virtual
# model name      : AMD EPYC Processor
#         Max Speed: 2000 MHz
# Connection to 5.223.50.71 closed.
```
