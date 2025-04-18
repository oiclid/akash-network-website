---
categories: ["Guides"]
tags: ["Storage"]
weight: 3
title: "Codes"
linkTitle: "Codex"
---


### Deploying Codex (codex.storage) on Akash using provider-services

#### Prerequisites
Ensure you have the following installed and configured:
- Akash CLI installed
- A funded Akash wallet

#### Step 1: Define Your SDL File
Create a file named `deploy.yaml` with the following content:

```yaml
---
version: "2.0"

endpoints: 
  nodeip:
    kind: ip

services:
  codex:
    image: codexstorage/nim-codex:0.1.6
    expose:
#    - port: 8080 # You probably don't want to expose the API globally:) Consider using reverse proxy with some auth?
#      as: 8080
#      to:
#        - global: true
#          ip: nodeip
    - port: 8070
      as: 8070
      proto: tcp
      to:
        - global: true
          ip: nodeip
    - port: 8090
      as: 8090
      proto: udp
      to:
        - global: true
          ip: nodeip
    env:
      - DATA_DIR=/codex/data
      - PRIV_KEY=
      - CODEX_NAT=
      - CODEX_LOG_LEVEL=debug
      - CODEX_BLOCK_TTL=30d 
    command:
    - sh
    - -c
    args:
    - >
        if [ -z "${CODEX_NAT}" ]; then
          echo "Missing \$CODEX_NAT=public_ip_of_the_deplyoment"
          sleep 120
          exit 1
        fi

        mkdir -p ${DATA_DIR}
        chmod 0700 ${DATA_DIR}

        quota=$(df --output=size --block-size=1 /codex | tail -1)

        exec /docker-entrypoint.sh codex \
          --bootstrap-node=spr:CiUIAhIhAiJvIcA_ZwPZ9ugVKDbmqwhJZaig5zKyLiuaicRcCGqLEgIDARo8CicAJQgCEiECIm8hwD9nA9n26BUoNuarCEllqKDnMrIuK5qJxFwIaosQ3d6esAYaCwoJBJ_f8zKRAnU6KkYwRAIgM0MvWNJL296kJ9gWvfatfmVvT-A7O2s8Mxp8l9c8EW0CIC-h-H-jBVSgFjg3Eny2u33qF7BDnWFzo7fGfZ7_qc9P \
          --bootstrap-node=spr:CiUIAhIhAyUvcPkKoGE7-gh84RmKIPHJPdsX5Ugm_IHVJgF-Mmu_EgIDARo8CicAJQgCEiEDJS9w-QqgYTv6CHzhGYog8ck92xflSCb8gdUmAX4ya78QoemesAYaCwoJBES39Q2RAnVOKkYwRAIgLi3rouyaZFS_Uilx8k99ySdQCP1tsmLR21tDb9p8LcgCIG30o5YnEooQ1n6tgm9fCT7s53k6XlxyeSkD_uIO9mb3 \
          --bootstrap-node=spr:CiUIAhIhA6_j28xa--PvvOUxH10wKEm9feXEKJIK3Z9JQ5xXgSD9EgIDARo8CicAJQgCEiEDr-PbzFr74--85TEfXTAoSb195cQokgrdn0lDnFeBIP0QzOGesAYaCwoJBK6Kf1-RAnVEKkcwRQIhAPUH5nQrqG4OW86JQWphdSdnPA98ErQ0hL9OZH9a4e5kAiBBZmUl9KnhSOiDgU3_hvjXrXZXoMxhGuZ92_rk30sNDA \
          --bootstrap-node=spr:CiUIAhIhA7E4DEMer8nUOIUSaNPA4z6x0n9Xaknd28Cfw9S2-cCeEgIDARo8CicAJQgCEiEDsTgMQx6vydQ4hRJo08DjPrHSf1dqSd3bwJ_D1Lb5wJ4Qt_CesAYaCwoJBEDhWZORAnVYKkYwRAIgFNzhnftocLlVHJl1onuhbSUM7MysXPV6dawHAA0DZNsCIDRVu9gnPTH5UkcRXLtt7MLHCo4-DL-RCMyTcMxYBXL0 \
          --bootstrap-node=spr:CiUIAhIhAzZn3JmJab46BNjadVnLNQKbhnN3eYxwqpteKYY32SbOEgIDARo8CicAJQgCEiEDNmfcmYlpvjoE2Np1Wcs1ApuGc3d5jHCqm14phjfZJs4QrvWesAYaCwoJBKpA-TaRAnViKkcwRQIhANuMmZDD2c25xzTbKSirEpkZYoxbq-FU_lpI0K0e4mIVAiBfQX4yR47h1LCnHznXgDs6xx5DLO5q3lUcicqUeaqGeg \
          --bootstrap-node=spr:CiUIAhIhAgybmRwboqDdUJjeZrzh43sn5mp8jt6ENIb08tLn4x01EgIDARo8CicAJQgCEiECDJuZHBuioN1QmN5mvOHjeyfmanyO3oQ0hvTy0ufjHTUQh4ifsAYaCwoJBI_0zSiRAnVsKkcwRQIhAJCb_z0E3RsnQrEePdJzMSQrmn_ooHv6mbw1DOh5IbVNAiBbBJrWR8eBV6ftzMd6ofa5khNA2h88OBhMqHCIzSjCeA \
          --bootstrap-node=spr:CiUIAhIhAntGLadpfuBCD9XXfiN_43-V3L5VWgFCXxg4a8uhDdnYEgIDARo8CicAJQgCEiECe0Ytp2l-4EIP1dd-I3_jf5XcvlVaAUJfGDhry6EN2dgQsIufsAYaCwoJBNEmoCiRAnV2KkYwRAIgXO3bzd5VF8jLZG8r7dcLJ_FnQBYp1BcxrOvovEa40acCIDhQ14eJRoPwJ6GKgqOkXdaFAsoszl-HIRzYcXKeb7D9 \
          --storage-quota=${quota} \
          --data-dir=${DATA_DIR} \
          --api-port=8080 \
          --api-bindaddr=0.0.0.0 \
          --disc-port=8090 \
          --listen-addrs=/ip4/0.0.0.0/tcp/8070 \
          persistence \
          --eth-provider=https://rpc.testnet.codex.storage

    params:
      storage:
        codex:
          mount: /codex
          readOnly: false
 
profiles:
  compute:
    codex:
      resources:
        cpu:
          units: 2
        memory:
          size: 4Gi
        storage:
          - size: 1Gi
          - size: 10Gi
            name: codex
            attributes:
              persistent: true
              class: beta3
    
  placement:
    dcloud:
      pricing:
        codex:
          denom: uakt
          amount: 100000

deployment:
  codex:
    dcloud:
      profile: codex
      count: 1 
```

#### Step 2: Submit the Deployment
Run the following command to create the deployment:
```bash
provider-services tx deployment create deploy.yaml --from <your-wallet>
```

#### Step 3: Find and Accept a Bid
List the available bids:
```bash
provider-services query market bid list --owner <your-wallet>
```
Find a suitable bid and accept it:
```bash
provider-services tx market lease create --owner <your-wallet> --dseq <deployment-sequence> --oseq 1 --gseq 1 --provider <provider-address>
```

#### Step 4: Retrieve Service Endpoint
Once the deployment is active, get the lease status:
```bash
provider-services query market lease list --owner <your-wallet>
```
Find the service URI and access Codex at the displayed endpoint.

#### Step 5: Verify Deployment
Run the following to check logs and verify that Codex is running:
```bash
provider-services query lease logs --dseq <deployment-sequence> --oseq 1 --gseq 1 --provider <provider-address>
```
You should now have Codex running on Akash!

#### Step 6: Managing the Deployment
To update your deployment:
```bash
provider-services tx deployment update deploy.yaml --from <your-wallet>
```
To close your deployment:
```bash
provider-services tx deployment close --from <your-wallet> --dseq <deployment-sequence>
```

This guide ensures a smooth deployment of Codex on Akash while using provider-services instead of Akash CLI. If you need further adjustments or have any questions, feel free to ask!