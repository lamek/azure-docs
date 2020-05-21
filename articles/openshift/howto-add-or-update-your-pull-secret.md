---
title: Add or update your Red Hat pull secret
description: Add or update your Red Hat pull secret on existing 4.x ARO clusters
author: sakthi-vetrivel
ms.author: suvetriv
ms.service: container-service
ms.topic: conceptual
ms.date: 05/21/2020
keywords: pull secret, aro, openshift, red hat
#Customer intent: As a customer, I want to add or update my pull secret on an existing 4.x ARO cluster.
---

# Add or update your Red Hat pull secret

This guide covers adding or updating your Red Hat pull secret for an existing Azure Red Hat OpenShift 4.x cluster.

If you are creating your cluster for the first time, then you can add your pull secret in the same command you use to create your cluster. See the [Create an Azure Red Hat OpenShift 4 cluster](tutorial-create-cluster.md#get-a-red-hat-pull-secret-optional) tutorial for more details.

## Before you begin

Ensure that you have administrator access to your cluster.

Follow the steps in the [Create an Azure Red Hat OpenShift 4 cluster](tutorial-create-cluster.md#get-a-red-hat-pull-secret-optional) to get your Red Hat pull secret.

## Add your pull secret to your cluster.



1. Fetch the secret named `pull-secret` in the openshift-config namespace.  This can be done by following this command:
oc get secrets pull-secret -o template='{{index .data ".dockerconfigjson"}}' | base64 -d

You can also clean up the json output by sending it to the jq tool or any other json parser. 

An example of using jq:
oc get secrets pull-secret -o template='{{index .data ".dockerconfigjson"}}' | base64 -d |jq

An example of using the Python mjson package:
python ex: oc get secrets pull-secret -o template='{{index .data ".dockerconfigjson"}}' | base64 -d | python -mjson.tool

2. Save this output to a separate file.
oc get secrets pull-secret -o template='{{index .data ".dockerconfigjson"}}' | base64 -d |jq > pull-secret.json

3. Edit the file by adding in an entry for the registry:
{
    {"auths:"
...
         "registry.redhat.io": {
            "auth": "<key>",
            "email": "testuser@domain.com"
         }
    }
}

4. Ensure the the file is valid json. There are many ways to validate your json. The following example uses jq:
cat pull-secret.json | jq

> NOTE
> If an error is in the file it can be seen `parse error`.

5. Run the following update command:
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=./pull-secret.json

Once the secret is set, Operator Hub should have access to the Red Hat certified operators.

## Validate that your secret is working

Run the the following test to validate that your secret has been added and is working correctly.
