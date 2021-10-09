---
layout: post
title:  "In case you want to run Polygon Node on Raspbery PI..."
date:   2021-10-09 02:41:49 -0500
categories: polygon-sdk raspberry-pi
---

## Configure polygon sdk

Checkout polygon-sdk sources

```bash
gh repo clone 0xPolygon/polygon-sdk
```

Following crosscompile command executed on Mac OS will produce a polygon-sdk binary that will run on RPI device configured with Ubuntu Core 20 64 bit

```bash
env GOOS=linux GOARCH=arm64  go build
```

## Raspberry Pi Setup

Follow official Ubuntu [installation steps for Raspberry Pi](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview)

