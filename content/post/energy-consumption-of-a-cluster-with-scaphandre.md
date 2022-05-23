+++
title = "Energy consumption of a Kubernetes cluster using Scaphandre"
description = "Measuring energy consumption of a Kubernetes cluster using Scaphandre."
tags = [
    "kubernetes",
    "rust",
    "sustainability",
]
date = 2022-05-22T09:00:00Z
author = "Ross Fairbanks"
+++

![Wind turbine with sunflowers](/images/wind-turbine-with-sunflowers.jpeg)

[Green Software Engineering](https://greensoftware.foundation/articles/what-is-green-software) is a new and evolving branch of software engineering for building more sustainable systems. Using these types of techniques the tech sector can play its part in tackling the climate crisis and reducing our dependence on fossil fuels.

For running software [electricity](https://principles.green/principles/electricity/) consumption is a key metric. Of course the source of that electricity and the carbon emissions [embodied](https://principles.green/principles/embodied-carbon/) in the hardware are also important. However even if the electricity sourced is 100% renewable consuming less electricity frees it up to be used for other purposes.

[Scaphandre](https://github.com/hubblo-org/scaphandre) is a monitoring agent that provides energy consumption metrics. The project was founded by Benoit Petit in October 2020. The naming comes from "heavy diving suit" in French. I've been fortunate to be able to help with the Kubernetes integration which this post focuses on and its given me the chance to learn some Rust. Scapandre is developed in Rust to keep the footprint of the monitoring as low as possible.

However Kubernetes isn't required. Scaphandre will work on any bare metal server providing the CPU and kernel has RAPL support. A great example is this [post](https://superuser.openstack.org/articles/environmental-reporting-dashboards-for-openstack-from-bbc-rd/) on how BBC R&D uses it and the UK National Grid [Carbon Intensity API](https://carbonintensity.org.uk/) to monitor their OpenStack data centres.

RAPL (Running Average Power Limit) is a technology developed by Intel that is included in most Intel and AMD CPUs produced after 2012. In some cases it can also measure DRAM energy usage. The `powercap` part of the name comes from the PowerCap Linux kernel module. This writes the energy consumption data to files in `/sys/class/powercap`. This directory and `/proc` are mounted by Scaphandre and used as the source of the metrics. The [docs](https://hubblo-org.github.io/scaphandre-documentation/explanations/how-scaph-computes-per-process-power-consumption.html) have a much deeper explanation of how the per process consumption is calculated.

For outputs Scaphandre has the concept of exporters. In this post we'll focus on the Prometheus exporter but other options are JSON, Riemann and Warp 10. The [Qemu exporter](https://hubblo-org.github.io/scaphandre-documentation/references/exporter-qemu.html) is a special case. It runs on bare metal hosts and can expose metrics to Qemu/KVM VMs running on the host.

Metrics are exposed at the host, socket and process level. For Kubernetes workloads these processes will be containers running on a node. The raw metric includes the PID of the container. However complex applications may consist of multiple containers. So we add the pod name and namespace each container belongs to. With these extra labels we can link to any other metadata we need in Prometheus.

![Prometheus query for Scaphandre data](/images/scaphandre-promql.png)

To do this when Scaphandre is deployed as a daemon set with the `--containers` flag it will open an in-cluster connection to list the pods on the node. To link processes to pods the cgroup file in `/proc/PID/cgroup` is parsed to get the container ID. 

There are some challenges here as the format of the cgroup file depends on the [cgroup driver](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/) used by the kubelet. We're working to support all drivers but if you encounter issues please open an issue.

![Grafana dashboard with Scaphandre data](/images/scaphandre-dashboard.png)

If you want to try out Scaphandre on a bare metal cluster there is a [tutorial](https://hubblo-org.github.io/scaphandre-documentation/tutorials/kubernetes.html) for installing it via its Helm chart along with Prometheus and Grafana to visualize the energy consumption of your cluster. Just be sure to check the [compatibility](https://hubblo-org.github.io/scaphandre-documentation/compatibility.html) docs to ensure your CPU and kernel is supported. 

There is lots planned for the project including an estimation based sensor that will work with VMs including public cloud instances. So watch this space! Or even better, get in touch if you're interested in [contributing](https://hubblo-org.github.io/scaphandre-documentation/contributing.html).

Photo by [Gustavo Quepón](https://unsplash.com/es/@unandalusgus) on [Unsplash](https://unsplash.com/es/fotos/pF_2lrjWiJE).
