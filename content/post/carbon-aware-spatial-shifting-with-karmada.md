---
author: Ross Fairbanks
cover:
  alt: Electricity transmission lines
  image: https://rossfairbanks.com/images/tranmission-lines.jpg
date: 2023-07-12T09:00:00.000Z
description: >-
  Adding spatial carbon awareness to Kubernetes workloads using Karmada.
tags:
  - carbon-awareness
  - karmada
  - kubernetes
  - sustainability
title: Carbon aware spatial shifting of Kubernetes workloads using Karmada
---

In my last [post](https://rossfairbanks.com/2023/06/05/carbon-aware-temporal-shifting-with-keda/),
I looked at temporal shifting of Kubernetes workloads using [KEDA](https://keda.sh)
and the [carbon-aware-keda-operator](https://github.com/Azure/carbon-aware-keda-operator) from Microsoft. With this approach,
non-time sensitive workloads can be run at times when the [carbon intensity](https://learn.greensoftware.foundation/carbon-awareness/#carbon-intensity)
of the electricity grid being used is lowest.

In this post we'll look at [spatial shifting](https://learn.greensoftware.foundation/carbon-awareness/#spatial-shifting),
which involves shifting workloads to physical locations where the grid carbon intensity is lower. This allows
workloads that are more time sensitive to be made carbon aware. It also enables a "follow the sun" model
to be used to move workloads across different geographies depending on how much renewable energy is available.

However, as with temporal shifting, there may be constraints that prevent spatial shifting. Such as regulatory
requirements that data must be stored and processed in certain countries. Data gravity can be another constraint.
For example, when training a machine learning model if it needs to fetch large quantities of data from another
region this may slow the job, increase costs, and there will be carbon emissions from the network transfer.

This is an area the community has been looking at for some time. Including the [paper](https://ceur-ws.org/Vol-2382/ICT4S2019_paper_28.pdf)
"A Low Carbon Kubernetes Scheduler" from 2019. I proposed a possible solution using KubeFed in a [blog post](https://www.thegreenwebfoundation.org/news/carbon-aware-scheduling-on-nomad-and-kubernetes/) I wrote for the Green Web Foundation in 2022.

KubeFed has now been [retired](https://www.cncf.io/blog/2022/09/26/karmada-and-open-cluster-management-two-new-approaches-to-the-multicluster-fleet-management-challenge/)
and replaced by two projects, Karmada and Open Cluster Management.

## Karmada

[Karmada](https://karmada.io) runs in a control plane cluster and consists of a series of controllers and a Karmada API Server as well
as the Kubernetes API Server. Multiple member clusters can then be registered with Karmada.

![Karmada architecture diagram](/images/karmada-architecture.png)

To schedule resources in the member clusters the resource templates are created in the Karmada API server. These can be built-in objects
like configmaps or custom resources and CRDs (Custom Resource Definitions). No pods will be scheduled for these template resources in the control plane cluster.

The user then defines a Propagation Policy with which resources to select and where to place them. The member clusters are
selected by setting the cluster affinities with the names of clusters. Here is an example policy for an nginx deployment.

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames:
      - member1
      - member2
    replicaScheduling:
      replicaSchedulingType: Divided
```

The easiest way to try Karmada is to follow their [quick start](https://github.com/karmada-io/karmada#quick-start)
which creates a control plane cluster and 3 member clusters using [kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker).
Just ensure your container engine has sufficient resources for 4 kind clusters!

## carbon-aware-karmada-operator

After trying out Karmada I think it has a lot of potential to be used for spatial shifting of workloads.
So I've created a proof of concept [carbon-aware-karmada-operator](https://github.com/rossf7/carbon-aware-karmada-operator).
The design follows the standard operator pattern of extending the Kubernetes API with a CRD and
running a controller in the cluster that watches for these custom resources.

The CRD is called CarbonAwareKarmadaPolicy and here is an example resource.

```yaml
apiVersion: carbonawarepolicy.rossf7.github.io/v1alpha1
kind: CarbonAwareKarmadaPolicy
metadata:
  name: nginx-policy
spec:
  clusterLocations:
  - name: member1
    location: DE
  - name: member2
    location: ES
  - name: member2
    location: FR
  desiredClusters: 2
  karmadaTarget: propagationpolicies.policy.karmada.io
  karmadaTargetRef:
    name: nginx-propagation
    namespace: default
```

For each member cluster, we include its name and the location code used by the carbon intensity API. In this case, it's
the country codes where the clusters are located but it could also be the name of the electricity grid e.g. `CAISO_NORTH`
for Northern California. See the next section for more details on the carbon intensity integration.

We also specify how many member clusters are desired and the propagation policy to update to set the cluster affinities.

Right now the scheduling logic is very basic. The operator ranks each member cluster by carbon intensity and selects the
desired number of clusters with the lowest values. Every 5 minutes the operator will check if there is new carbon intensity
data and repeat the ranking.

## grid-intensity-go

To access the carbon intensity data the operator uses the [grid-intensity-go](https://github.com/thegreenwebfoundation/grid-intensity-go)
library from the Green Web Foundation (which I'm a contributor to). As well as being a Go library it also has a CLI and a Prometheus exporter for accessing carbon intensity data. 

It supports multiple data providers including [Electricity Maps](https://www.electricitymaps.com/) and [WattTime](https://www.watttime.org/)
who provide data for multiple geographies. Both providers have commercial plans and free plans for non-commercial use. For WattTime their
free tier provides a relative value that can be used for ranking clusters but is less accurate than using absolute values.

Another important difference is Electricity Maps provides average carbon intensity and WattTime provides marginal carbon intensity.
See this [blog post](https://www.electricitymaps.com/blog/marginal-vs-average-real-time-decision-making) for more details.

grid-intensity-go provides `valid_from` and `valid_to` dates for each location and the operator has an in-memory cache so it only requests
new data when needed. This avoids hitting API rate limits and reduces the energy usage of the operator by reducing the number of API calls it makes.

## Status resource

The operator sets the status of the custom resource to show which clusters were selected
and the current carbon intensity data. 

```yaml
apiVersion: carbonaware.rossf7.github.io/v1alpha1
kind: CarbonAwareKarmadaPolicy
metadata:
  name: carbon-aware-nginx-policy
spec:
  ...
status:
  activeClusters:
  - member1
  - member2
  clusters:
  - carbonIntensity:
      units: gCO2e per kWh
      validFrom: "2023-07-12T11:00:00Z"
      validTo: "2023-07-12T12:00:00Z"
      value: "55.00"
    location: FR
    name: member1
  - carbonIntensity:
      units: gCO2e per kWh
      validFrom: "2023-07-12T11:00:00Z"
      validTo: "2023-07-12T12:00:00Z"
      value: "144.00"
    location: ES
    name: member2
  - carbonIntensity:
      units: gCO2e per kWh
      validFrom: "2023-07-12T11:00:00Z"
      validTo: "2023-07-12T12:00:00Z"
      value: "194.00"
    location: DE
    name: member3
  provider: ElectricityMap
```

The operator also emits Prometheus metrics per cluster with the carbon intensity and whether they are active.

```sh
curl -s http://localhost:8080/metrics | grep carbon_intensity
# HELP carbon_aware_karmada_operator_cluster_carbon_intensity Cluster carbon intensity
# TYPE carbon_aware_karmada_operator_cluster_carbon_intensity gauge
..._cluster_carbon_intensity{active="false",cluster="member3",location="DE"} 196
..._cluster_carbon_intensity{active="true",cluster="member1",location="FR"} 51
..._cluster_carbon_intensity{active="true",cluster="member2",location="ES"} 151
```

## Conclusion

This first version of the operator is a prototype to validate the idea and the
approach. The current scheduling logic is basic and could be improved.
It's also likely that more CRDs need to be managed. Karmada has support for
[Federated HPA](https://karmada.io/docs/tutorials/autoscaling-with-federatedhpa)
(Horizontal Pod Autoscaler) which would be very useful but uses additional CRDs.

If you want to try out the operator check out the [quick start](https://github.com/rossf7/carbon-aware-karmada-operator#quick-start).
Feedback is much appreciated and if you're interested in collaborating to develop the
operator further I'd be very interested.

#### Credits

- Photo by [Matthew Henry](https://unsplash.com/@matthewhenry) on [Unsplash](https://unsplash.com/photos/yETqkLnhsUI).
- [Karmada architecture diagram](https://github.com/karmada-io/karmada/blob/ccd8c3f863401ecf00bc8589e476bd5169d43d5f/docs/images/architecture.png)
