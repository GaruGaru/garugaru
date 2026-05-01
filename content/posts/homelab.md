+++
author = "GaruGaru"
title = "How I run my homelab"
date = "2026-04-20"
description = "Software and hardware fun building my homelab"
tags = [
"self-hosting",
    "infrastructure",
]
+++

I've been using my homelab since 1 year now, and I'm starting to be pretty happy about it and by happy I mean that I can stop thinkering and firefighting and actually starting to use it. 

As sole happy user and maintainer I'm going to write this blog post to describe the ideas and the steps behind the creation of my small homelab. 

# Hardware 

Let's start from the physical world, the main drivers behind my hardware choices were the following: 

* **Easiness of replacement**: No fancy, enterprise level or obscure harware, everything has to be standard and easly replaciable in case of failures.

* **Power consumption**: Since it will be running in my home I wanted hardware with low power consumption, which translates in lower temperatures and low noise.

* **Form Factor**: I wanted to fit my homelab in a small form factor which I can freely move around and doesn't take an entire room of space, small is beautiful ! 


Given those principles the hardware I'm currently using is:

* 2x Raspberry PI 5 - 8 GB memory
* 1x Raspberry PI 3 - 1 Gb memory 
* 1x Mini itx - intel N100 - 8GB memory - x3 512Gb ssd 
* 1 TP Link TL-SF1005D

## Rack 

![Rack](rack-0.jpg) 

Online we can find many pre-built racks coming in all sizes and shapes, as thinkerers we are never 100% happy about something which comes with all the bells and whistels,so of course here the solution was to build the rack my self.

I've started drawing my own 3d rack trying to fit everything in the smallest space possible, it took a while and more wasted filament that I'd like to admit but in the end it turned out pretty nice.


I'm a happy owner of a BambooLab A1 Mini, which is a great printer but it comes with a small build plate being able to print 18x18x18cm, this posed another challenge since I had to draw everything to be as modular as possible since I wasn't phisically able to print entire parts.

I've realized the simplest yet modular design that I was able to ( I'm not a 3d designer please forgive me ), where the basic idea was to have different **stackable** layers: each layer is composed by the stackable columns with supports to screw in masks so that it could either expose IO ports, fans or other mounts easly.

![Rack](rack-1.png)

## Power consumption and thermals.

The entire setup consumes about steady ~20W:
* 12W from the 3 RPIs
* 8W from the NAS running 3 ssd disks.

When in full load the power consumption can go up to 30-40W~ but this never happens in real world usage, so far I'm happy about the power consumption.

The RPIs are cooled using a single 80mm fan from Noctua and they are able to maintain a temperature of ~40-50°, the NAS is able to run completely fanless, I've already created a fan mount just in case for the future.

## Software

So what's actually running in the homelab and how do I manage it ?

Every machine in my cluster is running the same ubuntu LTS version, I've tried multiple fancier solutions such as using NixOS, CoreOS, pre-baked images or cloud-init/ignition but I wanted something as simple as possible and not too prone to change or being deprecated.

At the end I'm just installing the same commonly used packages, I want to be able to run the 2 command in 5 or 10 years and getting my homelab up again without worring about breaking changes or deprecated ways of doing yet-again-the-same-thing.

TLDR: the end result was: installing ubuntu LTS and performing the necessary provisioning and configuration using ansible without even relying on community playbooks.

### Kubernetes

I've made the choice of running kubernetes (using k3s) even if it went against the idea of simplicity, the main driver for this decision were me already having experience in running k8s clusters and the amount of tooling and deployment options for the software that I wanted to run, also see [kubernetes is not just for black friday](https://ergaster.org/posts/2025/07/09-kubernetes-black-friday]

I'm running a two nodes kubernetes cluster, 1 master, 1 worker running on the RPIs. 
The master node is running using a NVMe disk given the tendency of k8s of doing high IO on disk which resulted in many broken sd cards in the first year of running it.

The cluster bootstrap is performed by a simple ansible playbook which installs the necessary dependencies and setups the os and k8s cluster configuration.

#### Gitops 

Once the cluster is created I'm using the gitops tool [fluxcd](https://fluxcd.io/) to manage and keep all the kubernetes resources in sync with the wanted state. 

This way I'm able to update, install and maintain dependencies by just pushing to a git repository, flux takes care of deployment rollout without direct cluster access and prevent state from drifting. Which is also useful in case I need to recreate the cluster from scratch since I don't have to worry about forgetting some dependency between services and resoruces. 

The cluster configuration is stored in a [public github repository](https://github.com/garugaru/garu-kube), most of the application are deployment using the provided helm chart, but I'm trying to migrate as much as possible to [kustomize](https://kustomize.io/) because I find it a simpler and more kubernetes native way of handling resources.

#### Secrets management 

Given that the repository is public but I'm running some sensitive applications such as Immich for photo storage and some databases, the secrets are encrypted and managed using [sealed secrets](https://github.com/bitnami-labs/sealed-secrets). 
The encryption key is created once when the cluster bootstrapped and then used to decrypt the secrets cross all services.

#### Persistence

My home lab is running many stateful applications, some of them also require high(ish) performance storage like postgres databases, immich for photo storage and navidrome for music streaming.

There are many storage solutions running on top of kubernetes but I wanted to **separate compute from storage** 
 as much as possible, that I could experiment, scale or even completely destroy the 'compute' side of my homelab without losing important data.

The final solution was to use my ZFS NAS as storage, exposing ISCSI volumes to my kubernetes cluster which support [mounting NFS / ISCSI volumes natively](https://kubernetes.io/docs/concepts/storage/persistent-volumes/), this also enables transparent storage mounting so that stateful workloads can be moved between nodes without any issue.

This way I'm able to store and persist data using high performance and redunded NAS disks, while keeping the volume management simple integrating directly with the kubernetes apis.
Relying on ISCSI disks instead of NFS not only yields better performance but it's actually the only safe way to run postgres, sqlite and other systems which expect to perform IO on block level and not on file level.


### NAS

The NAS, which stands at the top level of my rack, is a Intel N100 mini ITX board with 3 ssd disks attached, 1 for the os and the other 2 are mirriored as part of the ZFS Pool. 

Since I wanted simplicity and control over the storage aspect I'm not using any solution like TrueNAS or Unraid, just plain ZFS configured by an ansible playbook. 

ZFS mirriors two SSD disks and expose volumes via ISCSI interface for the kubernetes cluster. 


#### Backups

Having redundant copies of your data is good, but having the copies phisically in the same location running attached on the same machine is not your best bet, for this reason I also have daily incremental backups of the ZFS volumes pushed on [blackbaze storage](https://www.backblaze.com/) which is pretty cheap and provide an s3 compatible api.


To perform the backups I wrote a small go program which runs in the NAS, it leverages the [ZFS incremental snapshot + send](https://docs.oracle.com/cd/E19253-01/819-5461/gbchx/index.html) features to take the backup and stream them directly to s3. 

To avoid having too many incremental parts in case of restore I also take a new full backup every 6 months, removing the older incremental backups saving storage cost and headaches. 


### What I'm Running 

Here's a brief list of what I'm running in my homelab: 

* [immich](https://immich.app/): photo manager, good google photo alternative with many nice features, good UI and native mobile app, very stable so far.

* [navidrome](https://www.navidrome.org/): simple and small audio streaming service, I use it to stream my music collection to multiple devices. 

* home assistant: (see below for more details)

* eink display api: small api which provide images for my DIY smart frame built using and ESP32 and Eink display 

![frame](eink-0.jpg) 

* Adguard: DNS level ad blocker for all devices in my LAN.

* surf condition bot: Telegram bot which detect good surf conditions and send me short webcam videos from surfs sport nearby me.

* cert-manager / external dns: automatically generates ssl certficates and updates DNS records when exposing new services.


### Home assistant

The lone RPI 3 is running home assistant installed directly as OS, using a zigbee dongle it is able to control zigbee devices in my home which are mainly smart switches / light bulbs. 

I decided to go for zigbee as I don't wanted to rely on thrid part apis or proprietary solutions to control my home devices, zigbee offers a standard protocol supported by many vendors which doesn't require internet or LAN connection to work. 

#### Valetudo

![Roomba](roomba-0.png) 

I've also modified my smart vacuum cleaner, which apparently is running Tina Linux installing [valetudo](https://valetudo.cloud/) on it, Valetudo is a nice software which allows you to disconnect your smart vacuum from the internet while keeping the original navigation capabilties. 
Using the MQTT plugin I'm able to control the vacuum cleaner directly from home assistant automations which is pretty handy. 

## Conclusions 

This is pretty much what I've done so far to build and maintain my homelab, I'll try to keep this article updated as I'll be using it as some sort of brain dump, thank you for reading this far ! 

