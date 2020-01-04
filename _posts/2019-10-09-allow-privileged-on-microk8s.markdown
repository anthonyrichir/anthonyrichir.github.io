---
layout: post
title:  "Allow privileged on Microk8s"
date:   2019-10-09 10:24:10 +0100
categories: kubernetes microk8s
---
I use Microk8s almost on a daily basis, with Helm, and if you have ever tried to deploy the official Elasticsearch chart from Elastic themselves on Microk8s, you have faced this issue were Helm says everything is installed but in the end, you don't get your Elasticsearch pods deployed by the operator.

It took me some time to figure out that it's actually the operator which is missing some privileges. And this can be very easily fixed by adapting the configuration of microk8s.

Just add `--allow-privileged=true` to the following config files.

* /var/snap/microk8s/current/args/kubelet
* /var/snap/microk8s/current/args/kube-apiserver

And finally restart your daemons.

```bash
sudo systemctl restart snap.microk8s.daemon-kubelet.service
sudo systemctl restart snap.microk8s.daemon-apiserver.service
```
