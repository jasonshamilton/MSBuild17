<!---title: Dependency Strategies when Containerizing Monolithic Architectures on ASF
 | Microsoft Docs
description: Best Practices and Platform Ecosystem for Containerization During a Digital Transformation
services: FabricApplicationGateway
documentationcenter: dev-center-name
author: GitHub-alias-of-only-one-author
manager: manager-alias

ms.service: required
ms.devlang: may be required
ms.topic: article
ms.tgt_pltfrm: may be required
ms.workload: required
ms.date: 04/23/2017
ms.author: jason.hamilton@wolterskluwer.com;semicolon separates two or more aliases

--->
# Overview

This document provides a summary view of how to define an effective Containerization Strategy as part of a multi-variable digital transformation.  Specifically it focuses on the journey from monolithic datacenter footprints to Hybrid cloud models that support an end goal of a public cloud microservice architecture for large distributed systems.

It maps out a path to support the existing software footprint, while moving towards the end goal in a step wise manner.  It focuses on accomplishing multiple targets and goals with containerization both enabling lift and shift and a transformation scenarios.
The following is a summary viewpoint on the transformation targets, the approaches taken, and how containerization plays a strategic role using Azure Service Fabric.

# Transformation Targets

1. Moving from a Monolith Single Solution Build to a CICD model using VSTS
    * Targeted as part of the strategy to prepare an existing code base for containerization at the subsystem level with an end game of continuous development and integration benefits
    * Enables a preparation and cleanup of code in a distributed system for containerization
2. Moving away from a Monolith API Set to Microservices using Azure API Management and Swagger
    * Targeted as a compute and system level transformation that allows with heavy client dependencies to an orchestrated microservice model that supports existing datacenters and new cloud hosting
    * Enables an API Mapping Dependency Discovery and Definition to define highest value transformation targets and sequence and provides roadmap for the subsystem definition and boundaries for containerization
3. Moving from Datacenter to Public Cloud Hosting
    * Targeted as a model to move various elements of the system within the technology stack in an iterative manner to allow the risk amount of risk to value with regarding to timing and rhythms of a business.
    * Enables subsystems to be containerized vs a monolithic lift and shift


# Technology Example Scenario

The following summarizes an example for discussion and backdrop.  These aren’t requirements in terms of mapping 1:1 for applicability.  Instead it paints a background context and perspective for the execution of the strategy.

- The existing system and associated architecture is primarily on the .Net Stack with a SQL backend
- The existing system has a large legacy multi-million line of code footprint build and expanded upon over the last decade
- The existing system scale year over year to a targeted number mainly through hardware (scale out) to accommodate growth in terms of load and capacity
- The existing system is highly transactional in nature and has stringent RPO and RTO requirements
- The system has several legacy client dependencies up and down the stack that require computational consideration for any strategy

# Strategy Requirements

Beyond just containerizing the existing workload, the strategy must accommodate both and asynchronous and concurrent activity to transform the existing architecture to a modern micro services model.  This is a key element to the containerization strategy. 

- The system must be transformed in a way that enables iterative and isolated refactor vs redesigning and redoing the system as a whole 
- The system must have demonstrated improvements in scale, near zero downtime, and supportability and maintainability.
- The strategy must support ability to update the system while using the system in production
- The strategy must support non digital transformation efforts to evolve and innovate the applications for customers
- The strategy must support iterative movements of components or subsystems of the application from Datacenter to Hybrid to Public Cloud

# Stepwise Approach

Based on the transformation needs 3 main playbooks were defined, Managing Client Dependencies, Distribution Model for Hybrid APIM, and Containerization Using Azure Service Fabric, to support the holistic Containerization Strategy

## Managing Client Dependencies
This section summarizes a summary playbook for using Nuget as a mechanism for dependency resolution

### Analyze, Identify, and Document 
Note: There are many tools which can enables this.  One example is NDepend.

- See: [Visualize Dependencies with NDepend](http://www.simpleorientedarchitecture.com/visualize-dependencies-with-ndepend/)
- See: [Package Management in Team Services and TFS](https://www.visualstudio.com/en-us/docs/package/overview) 

Dependency Playbook  | 
------------- |
Identify third party dependencies with their version number                     
Identify third party dependencies with their version number
Identify existing (intra) internal dependent binaries in the existing system    
Identify existing (inter) internal dependent binaries in the existing system  
Publish internal dependencies for each Nuget package
Create & check-in solution file with projects related to that package
Create Nuspec file to define contents of the package 
Update Build to publish Nuget package
Consume dependencies via Nuget
Resolve third-party dependencies via Nuget
Resolve internal dependencies via Nuget  


## Enabling a Distribution Model for Azure Hosted APIs

This section summarizes a summary playbook for using Publish APIs using WSDL / Swagger for Hybrid Hosting API Management

- See: [Publishing Metadata](https://msdn.microsoft.com/en-us/library/aa751951(v=vs.110).aspx)
- See: [SwashBuckle Plugin](https://github.com/domaindrivendev/Swashbuckle)
- See: [How to import the definition of an API with operations in Azure API Management](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-import-api) 

API Publication Playbook  | 
------------- |
Generate WSDL & Swagger files for services consumed by client applications and internal and external integrations 
For WSDL generation use native WCF Metadata publishing features
For Swagger generation use SwashBuckle plugin 
Ensure WSDL & Swagger file can be imported by Azure API Management
Create Axcess environment in Azure and point
Modify endpoint to APIM URLs in Dev environment and ensure that your service can be consumed via APIM


## Containerization Using Azure Service Fabric, Azure Container Registry, and Docker

This section summarizes a summary playbook for using Publish APIs using WSDL / Swagger for Hybrid Hosting API Management

ASF + Docker Containerization Playbook  | 
------------- |
Provision development environment for containerization
ASF Container Cluster
Azure Container Registry instance
Axcess environment for resolving database and other dependencies
Windows Server 2016 VM to create & test Dockerfile
Create Dockerfile per VSTS Repository
Each Docker file represents a container and will host all IIS sites & windows services contained in that repository
Instantiate container on Dev environment and ensure that services exposed by it works as expected
Create a CI Build for generating and publishing Docker image in WK Azure Container Registry

