---
layout: post
title:  "Hands on Falco"
date:   2025-11-06 18:00:00 +0200
year: 2025
categories: Kubernetes AKS Security
---

Hello!

Back to more k8s stuff again &#129299;

this time we'll have a look at Falco, a real time security monitoring solution.
We'll follow the below agenda:

1. Falco concepts
2. Falco on self-managed kubernetes
3. Falco on AKS
4. Writing custom rules
5. Using Falco sidekicks


Ok, let's get started!

## 1. Falco concepts

In the Kubernetes landscape, it's kind of a big issue to ensure the environments are secured.
While we have many different ways/tools avaialble to harden this security, it's noty a trivial problem to manage all those aspects.

When considering security monitoring, or shoulkd I say security posture &#129300;, there are a few tools that come to mind. [Falco](https://falco.org/docs/) is one of those.

Historically, Falco was created by [sysdig](https://www.sysdig.com/) but is now a CNCF graduated project.

Now about concepts, Falco is used to monitor environment and alert on abnormal behavior.
It relies on syscall to monitor the system activity.

In terms of architecture, we can find the following components from the documentation: 

- Event sources
- Rules
- Outputs
- Plugins
- Metrics

Falco monitoring is based on an analysis on different streams of events. The event sources mentionned earlier is the first compoent of Falco. AS mention previously, the default event source is syscall. 
Other event sources can be added through plugins. Note thatthe documentation reference a [registry](https://github.com/falcosecurity/plugins/blob/main/registry.yaml) for those plugins. One interesting plkugin (for me &#128527;) is the k8saudit-aks plugin, but that'll be for another time.

It is important to specify what Falco is not: A SIEM. IT is not able to correlate events from differents sources. 

Once we have identified the event sources, in our case, we'll stick to the default syscall, Falco can be configured with alert rules which are triggered depending on the events. And how do we configure those alerts? through a yaml definition.

The Falco rule is a yaml file, with three types of elements:

| Element | Description |
|-|-|
| Rules | |
| Macros | | 
| Lists | |


