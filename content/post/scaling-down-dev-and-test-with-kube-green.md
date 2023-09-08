---
author: Ross Fairbanks
cover:
  alt: Light bulbs and plants hanging from the ceiling.
  image: https://rossfairbanks.com/images/light-bulbs-and-plants.jpg
date: 2023-09-08T09:00:00.000Z
description: >-
  Using kube-green to suspend and resume non-production environments.
tags:
  - kubernetes
  - sustainability
title: Scaling down dev and test with kube-green
---

I think one of the hardest things about green software is knowing where to start. The scope is very wide, there are many possible solutions, and it's a new and evolving area. Some approaches I’ve written about, like carbon aware [demand shifting](https://rossfairbanks.com/2023/07/12/carbon-aware-spatial-shifting-with-karmada/) can be complex and not all tasks can be scaled in this way.

Measurement of your carbon emissions is a great place to start. You could use a dashboard from your cloud provider or an open source tool like [Cloud Carbon Footprint](https://www.cloudcarbonfootprint.org/). As without data you can’t tell where to focus your actions, or if they are helping or not. Yet, it’s natural to also want to take action to reduce your emissions. So what are some good first steps?

I like the [cloud zombies](https://hollycummins.com/cloud-zombies-qcon-london/) idea from Holly Cummins. Deleting unused infrastructure will reduce both costs and carbon emissions. For example it could be a cluster for a demo from 3 months ago that was never used since. OK, so you’ve gone on a zombie hunt and deleted some stuff, what’s next?

Scaling down apps in your dev, test and other non-production environments when they are not in use can be another good step. Again Holly has a great name for this which is [light switch ops](https://hollycummins.com/greener-applications-devoxx-uk/). To do this your apps must support being turned on and off quickly and reliably, like a light switch. Your apps may already support this, and if not it will make them better able to handle failure.

This is where [kube-green](https://github.com/kube-green/kube-green) comes in. It’s a Kubernetes operator developed by Davide Bianchi. It will scale down deployments and suspend cron jobs when they are not needed. kube-green follows the operator pattern. It extends the Kubernetes API with a CRD called SleepInfo and deploys a controller to do the scaling.

When a SleepInfo resource is created it will manage all cron jobs and deployments in its namespace unless you exclude them. Let’s look at an example.

```yaml
apiVersion: kube-green.com/v1alpha1
kind: SleepInfo
metadata:
  name: working-hours
spec:
  weekdays: "1-5"
  sleepAt: "20:00"
  wakeUpAt: "08:00"
  timeZone: "Europe/Rome"
  suspendCronJobs: true
  excludeRef:
    - apiVersion: "apps/v1"
      kind:       Deployment
      name:       api-gateway
```

This will suspend all cron jobs and scale down all deployments except api-gateway. This will happen at 20:00 each week day and wake up will happen at 08:00. During the weekend the pods will be scaled down.

There is one more step needed. In our cluster we need the cluster-autoscaler running.  This will scale down the cluster nodes and scale them back up again. Usually this isn’t possible for on premise clusters. Although kube-green will free up resources so they can be used for other tasks.

The docs have [install](https://kube-green.dev/docs/install/) instructions and a [tutorial](https://kube-green.dev/docs/tutorials/kind/) for getting started in a kind cluster. You can also see it in action in this great [video](https://www.youtube.com/watch?v=cMWZOB7-WtY
) from Saiyam Pathak on his Kubesimplify channel.
