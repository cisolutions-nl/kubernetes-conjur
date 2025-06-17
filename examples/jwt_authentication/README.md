# Conjur - Kubernetes/OpenShift Integration with ESO - JWT Authentication

This example demonstrates how to authenticate from the External Secrets Operator (ESO) to Conjur using **JWT-based authentication** in a Kubernetes environment.

## Overview

This example includes all the necessary resources to set up JWT authentication between ESO and Conjur:

- Conjur policy files that enable access from ESO via JWT
- Kubernetes resources for authentication using a namespace and service account
- ESO configuration to sync secrets from Conjur into Kubernetes

## Steps

1. **Deploy Conjur and configure the JWT authenticator.**
2. **Apply the Conjur policy files.**
3. **Apply the Kubernetes YAML files** to create the required resources:
   - Service account with projected token
   - CA bundle configuration for TLS
   - `SecretStore` and `ExternalSecret` definitions
