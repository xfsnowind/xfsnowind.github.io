---
title: "Learning Notes - Azure Challenge"
date: 2022-08-18T13:48:36+02:00
author: "Feng Xue"
toc: true
tags: ["Learning Notes", "Azure"]
draft: false
---

This blog is about the Azure developer challenge from Sparebanken Vest. I would like to write the learning notes here.

## Explore Azure App Service

### Target: 

> Learn about the key components of Azure App Service and how App Service can help you create, maintain, and deploy web apps more efficiently.

### Azure App Service

So what is Azure App Service? It's a Http-based service, which would server for web application, REST APIs and mobile backends. It can:
1. auto scale support
2. CI/CD support
3. Deployment slots -- support different deployment environment, like stage, prod
4. Linux support. But have some limitaions:
   1. Not support on Shared pricing tier;
   2. Cannot mix Windows and Linux in same App service plan;
   3. Cannot mix Windows and Linux apps in the same resource group after Jan 21, 2021;
   4. Azure Portal shows only working features 

### Azure App Service plans

An app runs in an *App Service plan* and each plan defines:
* Region (West US, North Euro, etc)
* Number of VM instances
* Size of VM instances
* Pricing tier (Free, Shared, Basic, Premium and etc)

And pricing tier decides what feature you can use:

* **Shared compute**: Both **Free** and **Shared** share the resource pools and can't scale out.
* **Dedicated compute**: The **Basic**, **Standard**, **Premium**, **PremiumV2**, and **PremiumV3** tiers run apps on dedicated Azure VMs. 
* **Isolated**: This tier runs dedicated Azure VMs on dedicated Azure Virtual Networks and provides the maximum scale-out capabilities.
* **Consumption**: This tier is only available to function apps. It scales the functions dynamically depending on workload.

In the **Free** and **Shared** tiers, app cannot scale out. And apps in the same App Service plan would share the same VM instances. 

When we add a new app, we need to understand the expected load for the new app. If it
* is resource intensive;
* needs resource in other geographical region;
* is scaled out independently from other apps;
, we can isolate it into a new App Service plan.

### Deploy

We can deploy App Service automatically from
* Azure DevOps
* Github
* Bitbucket

Or manually through:
* Git
* CLI - `az webapp up` would create a new App Service web app and deploy it;
* Zip deploy - Use `curl` or similar Http utility to send a ZIP to App Service;
* FTP/S

Deployment slots can be used when deploying a new production build.

### Authentication and Authorization in App Service

Built-in authentication can save time and effort with OpenID identity providers:
* Microsoft Identity Platform
* Facebook
* Google
* Twitter
* Any OpenID connect provider

The authentication and authorization will run in the same sandbox as application code and every incoming Http request would be handled with:
* authenticating with the specified provider
* validating, storing and refreshing tokens
* managing the authenticated session
* injecting identity information to request headers

Authentication flow:
1. Sign user in
2. Post authentication
3. Establish authenticated session
4. Serve authenticated content

In Azure portal, we can also configure App Service when the request is not authenticated:
1. Allow unauthenticated requests
2. Require authentication with Http 401/403

### Networking features

Sometimes we need to control the inbound and outbound network traffic. And features of inbound and outbound cannot be used to each other.

|  Inbound features | Outbound features |
|---|---|
| App-assigned address | Hybrid Connections |
| Access restrictions | Gateway-required virtual network integration |
| Service endpoints | Virtual network integration |
| Private endpoints |   |

But we can mix some features to solve the problems with a few exceptions.

|  Inbound use case | Feature |
|---|---|
| Support IP-based SSL needs for your app | App-assigned address |
| Support unshared dedicated inbound address for your app | App-assigned address |
| Restrict access to your app from a set of well-defined addresses | Access restrictions |


To Be Continued...