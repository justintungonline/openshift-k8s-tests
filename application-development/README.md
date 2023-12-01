# Openshift Kubernetes Test - Application Development

Following an Openshift Kubernetes workshop for developers on modern application development which can be run in [Red Hat Openshift playground](https://www.redhat.com/en/interactive-labs/openshift)

The workshop covers looking at an existing application and looking to modernizing the application. The work includes assessment, refactoring, API management, and creating intelligent applications using Red Hat AI Services.

## Architecture

- Existing Global Retail System Architecture is JavaEE, Oracle database, PostgreSQL, services on virtual machines and some on OpenShift
- Want to convert all component hosting into OpenShift container platform using Red Hat JBoss server. OpenShift on RHEL

## Workshop Steps and Technology

- Use the Migration Toolkit for Applications (MTA) to assess and prepare for app changes.
- Refactor the application, resolve issues and use recommendations.
- Deploy app to kubernetes (k8s)
- Secure APIs
- Create an intelligent application using data science, AI/ML models

The workshop environment has the following:

- Gitea to host the source code repositories
- OpenShift Virtualization to ultimately run the migrated PostgreSQL VM
- Migration Toolkit for Virtualization to facilitate the migration of the PostgreSQL VM to Red Hat OpenShift Container Platform
- OpenShift GitOps to manage the deployed services using a GitOps approach through ArgoCD
- OpenShift Pipelines to build the customer application from source code and deploy to the retail project using GiOps
- Migration Toolkit for Applications to modernize the customer service

## Workshop

Follow instructions and source repositories at: https://github.com/rh-mad-workshop
