---
title: "Troubleshooting Reconciliation"
date: 2023-07-19
tags: ["Tyk Operator", "Reconciliation", "Kubernetes"]
description: "How to check for reconciliation status using latestTransaction"
weight: 1
menu:
   main:
      parent: "Tyk Operator Reconciliation"
---

The Status subresource in Kubernetes is a specialized endpoint that allows developers and operators to retrieve the real-time status of a specific Kubernetes resource. By querying this subresource, users can efficiently access essential information about a resource's current state, conditions, and other relevant details without fetching the entire resource, simplifying monitoring and aiding in prompt decision-making and issue resolution.
With Tyk Operator v0.15.0, we introduce a new status subresource in APIDefinition CRD, called latestTransaction which holds information about reconciliation status.
Note that you can learn more about reconciliation here: Tyk Operator Reconciliation
The new status subresource latestTransaction consists of a couple of fields that show the latest result of the reconciliation. It mainly includes the following:
.status.latestTransaction.status: shows the status of the latest reconciliation, either Successful or Failed
.status.latestTransaction.time: shows the time of the latest reconciliation,
.status.latestTransaction.Error: shows the message of an error if observed in the latest transaction.
Consider the scenario when APIDefinition and SecurityPolicy are connected. Usually, APIDefinition cannot be deleted directly since it is protected by SecurityPolicy. The proper approach to remove an APIDefinition is to first remove the reference to the SecurityPolicy (either by deleting the SecurityPolicy CR or updating SecurityPolicy CR’s specification), and then remove the APIDefinition itself. However, if we directly delete this APIDefinition, Tyk Operator won’t delete the APIDefinition unless the link between SecurityPolicy and APIDefinition is removed.

$ kubectl delete tykapis httpbin 
apidefinition.tyk.tyk.io "httpbin" deleted 
^C%
After deleting APIDefinition, the operation hangs, and we suspect that something is wrong. 
Users might still look through the logs to comprehend the issue, as they did in the past, but they can now examine their APIDefinition’s status subresource to make their initial, speedy issue diagnosis.

$ kubectl get tykapis httpbin 
NAME      DOMAIN   LISTENPATH   PROXY.TARGETURL      ENABLED   STATUS
httpbin            /httpbin     http://httpbin.org   true      Failed
As seen in the STATUS column, something went wrong, and the STATUS is Failed. 
Before looking deeper into the Tyk Operator logs, we can briefly explain APIDefinition and determine what is wrong.

$ kubectl describe tykapis httpbin 
Name:         httpbin 
Namespace:    default 
API Version:  tyk.tyk.io/v1alpha1 
Kind:         ApiDefinition 
Metadata:
  ... 
Spec:
   ...
Status:
  api_id:                ZGVmYXVsdC9odHRwYmlu
  Latest CRD Spec Hash:  9169537376206027578
  Latest Transaction:
    Error:               unable to delete api due to security policy dependency=default/httpbin
    Status:              Failed
    Time:                2023-07-18T07:26:45Z
  Latest Tyk Spec Hash:  14558493065514264307
  linked_by_policies:
    Name:       httpbin
    Namespace:  default
or

$ kubectl get tykapis httpbin -o json | jq .status.latestTransaction
{
  "error": "unable to delete api due to security policy dependency=default/httpbin",
  "status": "Failed",
  "time": "2023-07-18T07:26:45Z"
}
Instead of digging into Tyk Operator's logs, we can now diagnose this issue simply by looking at the .status.latestTransaction field. As .status.latestTransaction.error implies, the error is related to SecurityPolicy dependency. For further information, we still suggest checking Tyk Operator logs.
