---
title: "Azure Api Management Automated Revisions"
date: 2021-07-28T23:41:05+03:00
draft: false
Summary: "Powershell script to deploy new WebAPI and release new API Management revision with zero downtime."
---

[Azure API Management](https://azure.microsoft.com/en-us/services/api-management/#overview) is designed to be an internet face for your API. The main reason that I started using it is the build-in trottling engine that should protect my API from unexpected workloads. But Azure API Management can do more. It tries to solve API versioning problems by introducing "[Versions](https://docs.microsoft.com/en-us/azure/api-management/api-management-versions)" and "[Revisions](https://docs.microsoft.com/en-us/azure/api-management/api-management-revisions)". The difference between those two is that "Version" is something that user decides to use by specifying URL part like "/v1" or "/v2" and "Revision" is the new version of particular version. By default API users are using the revision which developers forses them to use. Revisions is a good abstraction to use where you as a developer need to fix something in released API or introduce non-breaking changes. Here is my solution of how to automate new Azure API Management revisions delivery using powershell = your favorite CD tool:

{{< gist lAnubisl d04b2697f86a9c2d74a8264e2f361317 >}}