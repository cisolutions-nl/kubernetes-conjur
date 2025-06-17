# Conjur - Kubernetes/OpenShift Integration with ESO

This document describes how to integrate [CyberArk Secrets Management](https://www.cyberark.com/products/secrets-management/) (Conjur) with Kubernetes or OpenShift.  
The integration is done using the [External Secrets Operator (ESO)](https://external-secrets.io/latest/).

There are two supported authentication methods for accessing Conjur:

- **API key-based authentication**  
- **JWT-based authentication**

The [API key authentication example](examples/api-key_authentication/README.md) is located in the `examples/api-key_authentication/` directory.  
The [JWT authentication example](examples/jwt_authentication/README.md) is available in the `examples/jwt_authentication/` directory.

## Prerequisites

To follow this guide, you need a running instance of Conjur, either 
[Cloud](https://docs.cyberark.com/conjur-cloud/latest/en/Content/ConjurCloud/cl_ConjurCloudOverview.htm), 
[Enterprise](https://docs.cyberark.com/conjur-enterprise/latest/en/Content/Enterprise/WhatIsConjur.html?tocpath=Get%20started%7C_____1), 
or [Open Source](https://docs.cyberark.com/conjur-open-source/Latest/en/Content/Overview/Conjur-OSS-Suite-Overview.html).

You also need a running Kubernetes/OpenShift cluster. If you don't have one, you can use 
[Minikube](https://minikube.sigs.k8s.io/docs/start/) or [k3s](https://k3s.io/) to simulate a cluster locally or on your own server/VM.
In addition, your Kubernetes or OpenShift cluster must have the External Secrets Operator ([ESO](https://external-secrets.io/latest/)) installed.


The following tools are required on your local machine to follow this guide:

- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/): Kubernetes CLI
- [helm](https://helm.sh/docs/intro/install/): Kubernetes package manager
- [conjur](https://github.com/cyberark/conjur-cli-go): Conjur CLI

## Sources

The following sources have been used for this how-to:

- https://docs.cyberark.com/conjur-enterprise/latest/en/content/integrations/k8s-ocp/k8s-jwt-secrets-store-eso.htm
- https://external-secrets.io/latest/provider/conjur/
- https://github.com/conjurdemos/Accelerator-K8s-External-Secrets
