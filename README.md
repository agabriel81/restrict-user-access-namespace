# Restricting and monitoring users/groups permissions to terminal and pod debug capabilities

Limiting access to both terminal and/or pod debug capabilities can be compelling when a workload is exposing sensitive PII data which can be seen from users/groups.  
This repository is documenting 2 possible approaches on how to restrict and monitor the usage of both capabilities.  


## Requirements

The first solution doesn't require any special requirements because it leverages the standard built-in OpenShift RBAC capabilities.  
The second solution approach will require OpenShift 4.17+ as it leverages the ValidatingAdmissionPolicy / ValidatingAdmissionPolicyBinding included from Kubernetes 1.30+  
ValidatingWebhookConfiguration / WebhookConfiguration can be used as well but they won't be explored in this repository.  

## Clone the repo

~~~
$ git clone https://github.com/agabriel81/restrict-user-access-namespace
~~~

## Create a sample app in a test namespace

In OpenShift, all users will have the `self-provisioner` role and the `self-provisioners` cluster role binding by default to all authenticated users.   

~~~
$ oc new-project test
$ oc new-app --name nodejs-s2i https://github.com/nodeshift-starters/nodejs-rest-http -i nodejs:18-ubi8-minimal
~~~

And by default, the user creating the project, will get the `admin` permissions.  

## First approach: RBAC

In order to make sure that other users/groups joining the namespace cannot access either pod terminal and pod debug capabilites from both the Web Console and the `oc` CLI, we can assign the out-of-the-box `view` role.  
If we need more permissions such as the `edit` role, we can remove the `pod/exec` and the bare `pod/create` from the out-of-the-box `edit` role.  
In this approach, we will create a custom role by using the out-of-the-box `edit` role and removing the `pod/exec` and bare `pod/create` permissions.  
An example is provided here:  

~~~
$ oc project test
$ oc create -f roles/edit-without-terminal.yaml
$ oc adm policy add-role-to-user edit-without-terminal <your user> --role-namespace=test -n test
~~~

We can apply the same approach to the namespace's `admin` by removing the `pod/exec` and the bare `pod/create` from the out-of-the-box `admin` role.  
One of the downside of this approach is that the `cluster-admin` still has terminal feature enabled and it's not possible to remove it with the RBAC approach.  
We can implement a compensating control by collecting and analyzing the OpenShift audit logs.  
It's possible to see the live OpenShift audit logs with commands similar to:  

~~~
$ oc adm node-logs <one of your master nodes> --path=kube-apiserver/audit.log  | jq 'select(.requestURI | startswith("/api/v1/namespaces/test/")) | select(.objectRef.subresource == "exec")'
~~~

## Second approach: ValidatingAdmissionPolicy

Another approach could be to create a ValidatingAdmissionPolicy / ValidatingAdmissionPolicyBinding to restrict terminal and pod debug capabilities from namespaces where a particular label is assigned:  

~~~
$ oc label ns test debugpod=false
$ oc label ns test terminalaccess=false
$ for i in $(ls validatingadmissionpolicies); do oc create -f validatingadmissionpolicies/$i; done
~~~

## Test both approaches

Let's try both terminal and pod debug capabilites:  

~~~
$ oc project test
$ oc rsh <pod name> 
$ oc debug pod/<pod name>
~~~

For RBAC approach you'll get an output similar:

~~~
$ oc rsh nodejs-s2i-d97cdf6d6-5t4qg
Error from server (Forbidden): pods "nodejs-s2i-d97cdf6d6-5t4qg" is forbidden: User "user1" cannot create resource "pods/exec" in API group "" in the namespace "test"
$
$
$ oc debug pod/nodejs-s2i-d97cdf6d6-5t4qg   
Error from server (Forbidden): pods is forbidden: User "user1" cannot create resource "pods" in API group "" in the namespace "test"
$
~~~

For the ValidatingAdmissionPolicy approach you'll get an output similar:

~~~
$ oc rsh nodejs-s2i-d97cdf6d6-5t4qg
The pods "nodejs-s2i-d97cdf6d6-5t4qg" is invalid: : ValidatingAdmissionPolicy 'deny-terminal-access' with binding 'deny-terminal-access-binding' denied request: Accessing the pod terminal is not allowed
$
$
$ oc debug pod/nodejs-s2i-d97cdf6d6-5t4qg
The pods "nodejs-s2i-d97cdf6d6-5t4qg-debug-pr5tg" is invalid: : ValidatingAdmissionPolicy 'deny-pod-debug-access' with binding 'deny-pod-debug-access-binding' denied request: Creating debug pod is not allowed
$ 
~~~
