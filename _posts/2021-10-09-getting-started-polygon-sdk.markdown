---
layout: post
title:  "In case you want to start Polygon SDK cluster with node running on Raspbery PI..."
date:   2021-10-10 03:54:49 -0500
categories: polygon-sdk
---

## Configure polygon sdk

Checkout polygon-sdk sources

```bash
gh repo clone 0xPolygon/polygon-sdk
```

## Setup Polygon Validators - MUST HAVE AT LEAST FOUR

# Bootstrap few polygon-sdk nodes quickly following [official guide](https://sdk-docs.polygon.technology/docs/how-tos/howto-setup-ibft/howto-set-ibft-locally)

## Init nodes on local machine

```bash
polygon-sdk ibft init --data-dir data-dir1

[IBFT INIT]
Public key (address) = 0x0ab674FE72f4aB52dC99F424ecc086a6f6Cbce87
Node ID              = 16Uiu2HAmL5sEjLciHNc9VgJRGD8Mhe4KWrhTz8Kg7n2sazeDQmD9

polygon-sdk ibft init --data-dir data-dir2

[IBFT INIT]
Public key (address) = 0x0c98BB76ceAa2cA67eD745BE2C58755fb90a719f
Node ID              = 16Uiu2HAmKcQ59rELao3Kg9cuFzyQ6EHuFsjmwH3FfCMLx3swXqEv

polygon-sdk ibft init --data-dir data-dir3

[IBFT INIT]
Public key (address) = 0xFb8F2edC5d47342337bd4c72daB81FcD01d1D8E6
Node ID              = 16Uiu2HAm4ZxhPCjfFe1pcK1fU8732atXZkR2YjJRTe88jkCKJyMa
```

## Init node on Raspbery Pi

```bash
polygon-sdk ibft init --data-dir data-dir4

[IBFT INIT]
Public key (address) = 0x02BaF5c90C3576F378Fc6404af2af54e8D2AcCDf
Node ID              = 16Uiu2HAmDcfopFddSVe65AZ1ehhfmue1z25JGroRH3CYtobP89HN
```

## Create test account

Private key data for test accounts will be located in `data-dir1/keystore`

```bash
geth --datadir=./data-dir1 --password ./password.txt account new > account1.txt
```

Private key can be inspected with following command, private key shown here is used in hardhat project.

```bash
ethkey inspect --private --passwordfile password.txt data-dir1/keystore/UTC--2021-09-09T18-07-36.965882000Z--3bfc8a131be8ae818978f6fa57250bb3c8bbd919
Address:        0x3bFC8a131be8aE818978F6FA57250bB3c8BbD919
Public key:     0438de21eb96843d9cabb3d78a9376c6c25e5d1e435e1b2202ee5ca016bb007d4538352631474bc5f41e55e6c8b8579386fcf417310526d6b5bde4dc55f7f8e240
Private key:    XXXXXXXXXXXXX
```

Copy data-dir1/keystore contents to data-dir2, data-dir3 and Raspbery PI - data-dir4.

## Capture internal IPs of local machine (192.168.1.226), and remote node running on RPI (92.168.1.194)

## Generate genesis config

Preload one of the generated accounts with address from account1.txt, generated in previous step with some ETH

```bash
polygon-sdk genesis --consensus ibft --ibft-validator=0x0ab674FE72f4aB52dC99F424ecc086a6f6Cbce87 \
--ibft-validator=0x0c98BB76ceAa2cA67eD745BE2C58755fb90a719f \
--ibft-validator=0xFb8F2edC5d47342337bd4c72daB81FcD01d1D8E6 \
--ibft-validator=0x02BaF5c90C3576F378Fc6404af2af54e8D2AcCDf \
--bootnode=/ip4/192.168.1.226/tcp/10001/p2p/16Uiu2HAmL5sEjLciHNc9VgJRGD8Mhe4KWrhTz8Kg7n2sazeDQmD9 \
--premine=0x3bFC8a131be8aE818978F6FA57250bB3c8BbD919:1000000000000000000000
```

## Run nodes on local machine

```bash
polygon-sdk server --data-dir ./data-dir1 --chain genesis.json --grpc :10000 --libp2p 0.0.0.0:10001 --nat 192.168.1.226 --jsonrpc :10002 --seal --log-level DEBUG
polygon-sdk server --data-dir ./data-dir2 --chain genesis.json --grpc :20000 --libp2p 0.0.0.0:20001 --nat 192.168.1.226 --jsonrpc :20002 --seal --log-level DEBUG
polygon-sdk server --data-dir ./data-dir3 --chain genesis.json --grpc :30000 --libp2p 0.0.0.0:30001 --nat 192.168.1.226 --jsonrpc :30002 --seal --log-level DEBUG
```

## Run node on Raspbery PI

```bash
polygon-sdk server --data-dir ./data-dir5 --chain genesis.json --grpc :40000 --libp2p 0.0.0.0:40001 --nat 192.168.1.194 --jsonrpc :40002 --seal --log-level DEBUG
```
