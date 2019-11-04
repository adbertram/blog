---
title: Moving your on-premise Octopus install to Azure. 
description: How to move your On-Premise Octopus deploy instance to a highly-available Azure configuration. 
author: derek.campbell@octopus.com
visibility: private
published: 2019-11-14
metaImage: image.png
bannerImage: image.png
tags:
 - Product
---

![Image details](image.png)

## The Problem

Working as part of the Customer Success teams means we work with a range of customers from the smallest teams using Octopus in a single space to working with Global companies with diverse, distributed technical teams who want to work with Octopus to deploy Infrastructure, Databases and their packaged applications. In today's cloud-native world we see a lot of customers who want to move their Octopus workload from On-Premise data centres to Azure, and with our recent license changes which allows all new licenses the ability to be highly available, often the choice is to host Octopus in Azure. In this blog, we will cover how to move your existing Octopus instance to a highly available configuration in an Azure region.

[High Availability](https://octopus.com/docs/administration/high-availability) enables you to run multiple Octopus Deploy Servers, distributing load and tasks between them. This option has several benefits

### Existing Infrastructure

Most organizations using Octopus, are still running it in a single node configuration on a Virtual Machine hosted on a hypervisor with local storage. It's normally hosted on a dedicated Virtual Machine listening over HTTP or HTTPS using a [LetsEncrypt](https://letsencrypt.org/) SSL certificate or a SSL certificate generated by a [Certificate Authority](https://en.wikipedia.org/wiki/Certificate_authority) much like [GoDaddy](godaddy.com).

Unfortunately, [LetsEncrypt](https://letsencrypt.org/) SSL certificates are not supported automatically in a Highly-Available configuration, so if you are planning moving it to Azure, then you may need one from a [Certificate Authority](https://en.wikipedia.org/wiki/Certificate_authority).

If you're hosting on-Premise, you will likely have Octopus Confgured on the C or D Drive, under C:\Octopus or D:\Octopus with Logs, Artifacts, Packages and Task Logs hosted under folders locally or on a Network Attached Storage device.

### Proof of Concept vs Production move

Generally, when doing something for the first time, we recommend running a Proof of Concept before trying it on your production instance of Octopus. You will need to decide whether you are doing this as a Prood of Concept or in Production ahead of time.

There would be nothing wrong with lifting and shifting your existing Production instance and moving it to the cloud and get this working before then making it highly-available by adding in a second node, and a load balancer as long as you have already configured shared storage for Artifacts, Packages and TaskLogs. The benefit of this approach is that it would likely involve less testing and you can configure it to be highly available once your Octopus instance in Azure has been tested and verified.

## Prep work

There is going to be some preparation required ahead of your move and it will require some time where Octopus is not going to be able to run deployments as we should enable Maintenance Mode, drain all nodes, create a SQL Database backup, run a backup of your Artifacts, Packages and Tasklogs and then copy these up to Azure.

### Maintenance Mode

The first thing to do, would be to enable Maintenance Mode. n summary Maintenance Mode enables you to safely prepare your server for maintenance, allowing existing tasks to complete, and preventing changes you didn't expect.

To enable or disable Maintenance Mode, go to **{{Configuration,Maintenance}}**.

![Maintenance Mode Configuration](images/maintenance-mode.png)

Only users with the `Administer System` permission can enable/disable Maintenance Mode.

Once Octopus is in Maintenance Mode:

- Users with the `Administer System` permission can still do anything they want, just like normal. All other users are prevented from making changes, which includes queuing new deployments or other tasks.
- The task queue will still be processed:
  - Tasks which were already running will run through to completion.
  - Tasks which were already queued (including [scheduled deployments](/docs/deployment-process/releases/index.md#scheduling-a-deployment)) will be started and run through to completion.
  - System tasks will still be queued and execute at their scheduled intervals. These kinds of tasks can be ignored since they are designed to be safe to cancel at any point in time.

### Drain Nodes

You also have the ability to stop all tasks from being executed on Octopus, and this becomes more useful if you're running Octopus in a Highly-Available configuration but it is also useful if you want to stop Octopus Administrators from running any tasks.

For each node you have do the following:

1. Go to **{{Configuration > Nodes}}** and put the node into drain mode.
1. Wait for any remaining Octopus Tasks on that node to complete.
1. Stop the Octopus Server windows service on that node.

At this point, your Octopus server is ready to be moved to Azure and it will restore it in Azure with the nodes drained and if you enabled Maintenance Mode, then it will also have Maintenance mode enabled.

### Backup your Master Key

If you take one thing from this blog, please let it be this. Backup your Master Key, then back it up again, and then lastly, back it up once more.

Octopus [encrypts important and sensitive data](/docs/administration/security/data-encryption.md) using a master key. If you lose your master key, then you would lose:

- The Octopus Server X.509 certificate which is used for [Octopus to Tentacle communication](/docs/administration/security/octopus-tentacle-communication/index.md) - this means your Tentacles won't trust your Octopus Server any more.
- Sensitive variable values, wherever you have defined them.
- Sensitive values in your deployment processes, like the password for a custom IIS App Pool user account.
- Sensitive values in your deployment targets, like the password for creating [Offline Drops](/docs/infrastructure/deployment-targets/offline-package-drop.md).

To see your Master Key, log on your Octopus Server, open up Octopus Server Manager and select View master key, and save this to your server as a txt file and then back it up to a Password Manager much like [LastPass](http://lastpass.com/) or [KeyPass](https://keepass.info/). 

![Master Key backup](/masterkey.png)

## Azure Architecture

### Virtual Machines

### SQL Database

#### Azure SQL vs SQL Virtual Machine

### Shared storage

Octopus stores a number of files that are not suitable to store in the database. These include:

- NuGet packages used by the [built-in NuGet repository](https://octopus.com/docs/packaging-applications/package-repositories) inside Octopus. These packages can often be very large.
- Artifacts collected during a deployment. Teams using Octopus sometimes use this feature to collect large log files and other files from machines during a deployment.
- Task logs, which are text files that store all of the log output from deployments and other tasks.

As with the database, from the Octopus perspective, you'll simply tell the Octopus Servers where to store them as a file path within your operating system. Octopus doesn't really care what technology you use to present the shared storage, it could be a mapped network drive, or a UNC path to a file share. Each of these three types of data can be stored in a different place.

Whichever way you provide the shared storage, a few considerations to keep in mind:

- To Octopus, it needs to appear as a mapped network drive (e.g., D:\) or a UNC path to a file share (e.g., \\server\path)
- The service account that Octopus runs as needs full control over the directory
- Drives are mapped per-user, so you should map the drive using the same service account that Octopus is running under

#### Azure Files

If your Octopus Server is running in Microsoft Azure, there is really only one solution unless you have a [DFS Replica](https://docs.microsoft.com/en-us/windows-server/storage/dfs-replication/dfsr-overview) in Azure, and that solution is [Azure File Storage](https://docs.microsoft.com/en-us/azure/storage/files/storage-files-introduction) - it just presents a file share over SMB 3.0 that can be shared across all of your Octopus servers.

Once you have created your File Share you can mount the drive for use by Octopus. Remember, drives are mounted per user. Make sure to map a persistent network drive for the user account the Octopus Server is running under.

Run the below before installing Octopus.

````powershell
# Add the Authentication for the symbolic links. You can get this from the Azure Portal

cmdkey /add:octostorage.file.core.windows.net /user:Azure\octostorage /pass:XXXXXXXXXXXXXX

# Add Octopus folder to add symbolic links

New-Item -ItemType directory -Path C:\Octopus
New-Item -ItemType directory -Path C:\Octopus\Artifacts
New-Item -ItemType directory -Path C:\Octopus\Packages
New-Item -ItemType directory -Path C:\Octopus\TaskLogs

# Add the Symbolic Links. Do this before installing Octopus

mklink /D C:\Octopus\TaskLogs \\octostorage.file.core.windows.net\octoha\TaskLogs
mklink /D C:\Octopus\Artifacts \\octostorage.file.core.windows.net\octoha\Artifacts
mklink /D C:\Octopus\Packages \\octostorage.file.core.windows.net\octoha\Packages
````

Install Octopus and then run the below.

````powershell

# Set the path
& 'C:\Program Files\Octopus Deploy\Octopus\Octopus.Server.exe' path --artifacts "C:\Octopus\Artifacts"
& 'C:\Program Files\Octopus Deploy\Octopus\Octopus.Server.exe' path --taskLogs "C:\Octopus\TaskLogs"
& 'C:\Program Files\Octopus Deploy\Octopus\Octopus.Server.exe' path --nugetRepository "C:\Octopus\Packages"
````

### Load Balancer

This one took me a bit of time to make a selection as there are a few options for balancing your loads in Azure. It will depend entirely on your preference and what your Network team have a preference for as there are so many option, and we are only listing a few of these below.

* [Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview)
* [Azure Application Gateway](https://docs.microsoft.com/en-us/azure/application-gateway/overview)
* [Azure Load Balancer](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-overview)
* [Kemp LoadMaster](https://kemptechnologies.com/uk/solutions/microsoft-load-balancing/loadmaster-azure/)
* [F5 Big-IP Virtual Edition](https://www.f5.com/partners/technology-alliances/microsoft-azure)

My preference after evaluating the options on Azure

### Networking

### How to Test for a working cluster

## Conclusion