Kubernetes Replicator
===================

A Kubernetes controller that can be used to replicate secrets and configmaps across namespaces.

Source: https://github.com/mittwald/kubernetes-replicator

Directory Structure
-----------------

::

    .
    └── clusters/
        ├── base/
        │   ├── sources/
        │   │   └── mittwald.yaml            # HelmRepository
        │   └── infrastructure/
        │       └── replicator/
        │           ├── kustomization.yaml    # Base kustomization
        │           └── release.yaml          # HelmRelease
        └── stg/
            └── infrastructure/
                └── replicator/
                    └── kustomization.yaml    # Environment specific

Usage
------

replicator and flux clash as both want to modify/own the
resources.

The easiest way is to use push-base replication based on target labels:

- label the target namespace with::

    labels:
      replicator.<secret-ref>: "true"

- annotate the secret we want to replicate with::

    annotations:
      replicator.v1.mittwald.de/replicate-to-matching: replicator.<secret-ref>=true
    
Where <secret-ref> is any id you choose to identify your secret

Clearly you need to create separateli, and not in flux the source
secret. As a possible example for a docker registry::

  kubectl create secret docker-registry regcredsecret \
    --namespace g-iot-core \
    --docker-server=server \
    --docker-username=username \
    --docker-password=password

  kubectl annotate secret regcredsecret -n g-iot-core \
    replicator.v1.mittwald.de/replicate-to-matching: replicator.giot-docker-registry=true

or a simple secret for database credentials::

  kubectl create secret generic pg-credentials \
      --namespace=g-iot-core \
      --from-literal=password="pass" \
      --from-literal=username="user" 

  kubectl annotate secret pg-credentials -n g-iot-core \
    replicator.v1.mittwald.de/replicate-to-matching: replicator.giot-pg-credentials=true
