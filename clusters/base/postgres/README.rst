========================
PostgreSQL Cluster Setup
========================

This guide explains how to deploy a new CloudNativePG cluster using our Flux-based configuration.

Prerequisites
------------
- Access to the Kubernetes cluster
- Flux installed and configured
- ``kubectl`` CLI tool
- Access to the git repository

Step 1: Namespace Configuration
--------------------------------

Modify the kustomization.yaml to specify your target namespace::

    # clusters/stg/postgres/kustomization.yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    namespace: your-target-namespace    # Change this value
    resources:
      - ../../base/postgres
      - namespace.yaml

**Note**: If the namespace already exists:
  - Remove ``namespace.yaml`` from the resources list
  - Keep only the namespace specification in kustomization.yaml
  - Make sure you have the necessary permissions in the existing namespace

Step 2: Check HelmRepository Source
------------------------------------

Verify if the CloudNativePG HelmRepository already exists in flux-system namespace::

    kubectl get helmrepository cnpg -n flux-system

If it doesn't exist, it will be created by the source configuration in ``base/source/helmrepository-cnpg.yaml``.

Step 3: Create Required Secret
-------------------------------

Create the required secret in your target namespace using the template::

    # Example from secrets.yaml.template
    kubectl create secret generic pg-credentials \
      --namespace=your-target-namespace \
      --from-literal=POSTGRES_PASSWORD="your-password" \
      --from-literal=S3_SECRET_KEY="your-s3-key"

**Important**: Keep these credentials secure and never commit them to the repository.

Step 4: Apply and Verify
----------------------

1. Commit and push your changes
2. Wait for Flux to reconcile or force reconciliation::

       flux reconcile helmrelease cnpg -n your-target-namespace

3. Monitor the deployment::

       kubectl get pods -n your-target-namespace
       kubectl get postgresql -n your-target-namespace

Troubleshooting
-------------

Check Kustomization status::

    kubectl get kustomizations.kustomize.toolkit.fluxcd.io

Common errors:
- ``namespace not found``: Ensure namespace exists or is properly defined
- ``secret not found``: Verify secret creation in the correct namespace
- ``chart not found``: Check HelmRepository in flux-system namespace

Optional: Automatic Values Updates
-------------------------------

To automatically trigger updates when ConfigMaps change, consider adding ``dependsOn`` to your HelmRelease::

    # clusters/base/postgres/helmrelease.yaml
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: cnpg
    spec:
      dependsOn:
        - name: local-values
          kind: ConfigMap
        - name: base-values
          kind: ConfigMap
      # ... rest of the configuration

This is recommended when:
- Values change frequently
- Automatic updates are required
- Manual reconciliation should be avoided

Without this configuration, manual reconciliation is required after value changes::

    flux reconcile helmrelease cnpg -n your-target-namespace

Best Practices
------------

1. Always use version control for configuration changes
2. Test changes in a non-production environment first
3. Keep secrets separate from repository
4. Monitor cluster health after changes
5. Maintain backup configuration
