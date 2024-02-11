+++
author = "GaruGaru"
title = "5 Years of platform engineering, the Good, the Bad and the Messy"
date = "2024-02-09"
description = "Reflections on technologies and practices after 5 years"
tags = [
    "devops",
    "infrastructure",
]
+++

After 5 years I've left {{company name}}, during the years I had the opportunity to work and improve the core cloud infrastructure, covering topic such as scalabilty, bigdata, developer experience and much more. 

During my experience we tried different technologies, approaches and we achieved some big wins and, admittedly, made some regrettable choices.

It was quite a journey and I'd like to talk to you about the good and the bad decisions we've made and the rationale behind them.

## (too many) Services ❌

When scaling up and building features on top of the existing infrastructure, the standard approach we kept was to create new services, which usually required an API along with some ingestion system, and then integrate them into the current infrastructure using remote calls or queues. 

While this approach allowed us to add features quickly, it also got out of control pretty fast and showed its limits. Managing multiple services and repositories became difficult when performing basic debugging sessions, dependencies upgrades, and feature integrations. 

In hindsight, we should have taken more time thinking and planning how to integrate new features and products into the existing infrastructure, instead of creating new services without taking into account the complexity that we were introducing.

Interestingly, uber had a similar problem: https://www.uber.com/blog/microservice-architecture/
 

## Managed services ✅

Since the beginning, we made a firm decision to use only managed services.
This significantly helped us save engineering time, which was crucial as we were a small team tasked with building an entire infrastructure from scratch.

While it may not seem like the most cost-effective decision in the short term, avoiding the need to perform upgrades, provide high availability, conduct capacity planning, and handle general maintenance on services was totally worth it in the long run.

## Observability ✅

At some point we had almost 100 services deployed, many of them were internal or public facing apis which needed to communicate with other services to serve their requests.

As you may imagine, debugging distributed calls between different services and replicas using just logging and metrics was a slow, boring, and error-prone process, which really hindered our ability to narrow down issues and resolve them promptly.

In order to solve this issue we introduced distributed request tracing in all our api services using [open-telemetry](https://opentelemetry.io/) and [jaeger](https://www.jaegertracing.io/).

Having DRT in place immediately changed how fast and how we debugged our services; it also revealed some performance bottlenecks in the call graph of which we weren't even aware.    


## Kubernetes ✅

After less than 1 year, AWS ECS began to reveal its limitations. Primarily, we encountered difficulties in managing rollouts, configuring alerts, and, in general, some of the AWS integrations seemed somewhat hackish at times.

We made the big decision to migrate the infrastructure for both APIs and workers to Kubernetes, leveraging managed Kubernetes services (EKS).

The migration process went smoothly since we were already using containers and exposing our service via a managed API Gateway.

The usage of Kubernetes surely introduced some additional complexity, but it also enabled service deployments for different use cases, simpler integrations of monitoring, observability, and overall more control over the infrastructure.

Honorable mention to:
 * external-dns
 * flux ( gitops engine )
 * argo workflows 


## Gitops ✅

At {{company name}} we had to manage many different kubernetes clusters dedicated to specific workloads.
Every cluster had some specific configuration quirks along with common resources that were deployed cross cluster / region.

To address the configuration drift problem, we implemented GitOps principles using Flux.
This allowed us to effortlessly manage, deploy, and even perform rollbacks on various clusters in different regions by simply pushing changes to Git and trusting Flux and Kubernetes to reconcile the desired state.


## Unified compute and storage for datalake ❌

At {{company name}}, data was a core asset, thus, from the very beginning, we designed and implemented a data lake that acted as a source of truth, enabling data analysis, ETL, and API use cases.

Given the complexity of creating a data lake from scratch, we decided to use AWS Redshift, a managed data warehouse platform that allowed us to perform standard SQL queries while providing high-performance ingestion using AWS Firehose.

Once the use cases became numerous and the workloads too heavy, having both data and storage coupled together started to show its limitations, and scalability issues became clear.

We needed to scale for both analytical and interactive workloads (e.g., APIs vs. data scientist large analysis). The high-level design choice was to decouple the data storage from the compute.

To achieve this separation, we used AWS S3 as block storage along with Apache Iceberg as the table format, the interoperability of Apache Iceberg enabled many integrations and use cases.

Later, we introduced Apache Spark and Apache Flink for ETL and streaming workloads, integrated Trino for the read-heavy use cases such as APIs.

Being able to connect multiple applications and frameworks to the same data storage was a game-changer. We were not limited by a single data storage, and we also introduced many tools and integrations transparently and without overloading the single point of failure / bottleneck that was redshift.

## Non IaC native tools ❌

We decided to use eksctl for its simplicity in creating and managing our Kubernetes clusters by just using a yaml definition.  

Sadly, we regretted this decision quickly: eksctl was really difficult to integrate with the existing IaC tools such as terraform or CDK requiring continuous hacks and workarounds for smooth operations.

A seamless integration with other IaC tools was required since we wanted to be able to manage our kubernetes cluster consistenly with other resources such as vpcs, security groups, dns etc.

After a short period, we performed a migration to manage the infrastructure for both K8s and other resources using terraform with the aws providers and [some handy modules](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)


## No secret manager integration from the beginning ❌

When starting to develop a new infrastructure, it is easy to leave out some important aspects that may slow down development in later iterations.
This is especially true for security features, which are often overlooked, especially in startups. We made the mistake of not integrating a secret manager to store credentials and general secrets from the beginning.

This led to a situation where credentials were committed inside the repositories and scattered across different projects.

The fragmentation created by the many configuration methods used across different teams and languages made it difficult to perform tasks such as credential rotations and security assessments.


## No shared infrastructure decision making ❌

For the sake of speed and independence, every team was allowed to develop their own solutions and create their own cloud resources. However, many times, this created problems from a cloud spending perspective since the solutions weren't validated by the team responsible for spending. Developers were overlooking possible scalability and spending issues with the technical solutions

## No common resources tagging ❌

Resource tagging is important, especially when you have to to correlate cloud resources costs to actual products.

We made the mistake of not creating a standardized set of tags to apply to every resource, which would have helped us when doing cost optimization, analysis and allocation.


## GO Adoption ✅

Initially, the tech stack was Java/Spring Boot-based but the main pain points with Java + Spring Boot were the slow startup times, high memory usage, and large image sizes. 
While Java projects necessitated a significant amount of code and boilerplate, Go gained quick adoption by different teams once its performance and ergonomics became evident, quickly becoming the default language of choice. 
Java remained the language of choice for large, public-facing APIs due to its mature ecosystem, support for generics, and comprehensive tooling.


## Automated project scaffolding ✅

To accommodate the creation of new projects and ensure that every new repository adhered to best practices in terms of project structure, security, observability, and scalability, we automated the repository creation process. We provided developers with a set of predefined templates (e.g., java-api, go-worker, go-api, etc.).

By leveraging [Backstage](https://backstage.io/), developers were able to create, scaffold, and deploy (on multiple environments) a new project automatically, without having to configure anything or write all the needed boilerplate just to have a running service.