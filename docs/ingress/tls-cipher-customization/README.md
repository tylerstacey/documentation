# Configure ROSA/OSD to use custom TLS ciphers on the ingress controllers #

**Michael McNeill**

*24 August 2022*

This guide demonstrates how to properly patch the cluster ingress controllers, as well as ingress controllers created by the Custom Domain Operator. This functionality allows customers to modify the `tlsSecurityProfile` value on cluster ingress controllers. This guide will demonstrate how to apply a custom `tlsSecurityProfile`, a scoped service account (with the associated role and role binding), and a CronJob that the cipher changes are reapplied with 60 minutes (in the event that an ingress controller is recreated or modified).

## Before you Begin

Review the [OpenShift Documentation that explains the options for the `tlsSecurityProfile`](https://docs.openshift.com/container-platform/4.11/networking/ingress-operator.html#configuring-ingress-controller-tls). By default, ingress controllers are configured to use the `Intermediate` profile, which corresponds to the [Intermediate Mozilla profile](https://wiki.mozilla.org/Security/Server_Side_TLS#Intermediate_compatibility_.28recommended.29).

## 1. Create a service account for the CronJob to use

A service account allows our CronJob to directly access the cluster API, without using a regular user's credentials. To create a service account, run the following command:

```
oc create sa cron-ingress-patch-sa -n openshift-ingress-operator
```

## 2. Create a role and role binding that allows limited access to patch the ingress controllers

Role-based access control (RBAC) is critical to ensuring security inside your cluster. Creating a role allows us to provide scoped access to only the API resources we need within the cluster. To create the role, run the following command:

```
oc create role cron-ingress-patch-role --verb=get,patch,update --resource=ingresscontroller.operator.openshift.io -n openshift-ingress-operator
```

Once the role has been created, you need to bind the role to the service account using a role binding. To create the role binding, run the following command:

```
oc create rolebinding cron-ingress-patch-rolebinding --role=cron-ingress-patch-role --serviceaccount=openshift-ingress-operator:cron-ingress-patch-sa -n openshift-ingress-operator
```

## 3. Patch the ingress controller

> **Important note:** The examples provided below add an additional cipher to the ingress controller's `tlsSecurityProfile` to allow IE 11 access from Windows Server 2008 R2. You should modify this command to meet your specific business requirements. 

Before we create the CronJob, we first want to apply the `tlsSecurityProfile` configuration to validate our changes. This process depends on if you are using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html). 

### Clusters not using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html)

If you are only using the default ingress controller, and not using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html), you will run the following command to patch the ingress controller:

```
oc patch ingresscontroller/default -n openshift-ingress-operator --type=merge -p '{"spec":{"tlsSecurityProfile":{"type":"Custom","custom":{"ciphers":["TLS_AES_128_GCM_SHA256","TLS_AES_256_GCM_SHA384","ECDHE-ECDSA-AES128-GCM-SHA256","ECDHE-RSA-AES128-GCM-SHA256","ECDHE-ECDSA-AES256-GCM-SHA384","ECDHE-RSA-AES256-GCM-SHA384","ECDHE-ECDSA-CHACHA20-POLY1305","ECDHE-RSA-CHACHA20-POLY1305","DHE-RSA-AES128-GCM-SHA256","DHE-RSA-AES256-GCM-SHA384","TLS_CHACHA20_POLY1305_SHA256","TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA"],"minTLSVersion":"VersionTLS12"}}}}'
```

This patch will add the `TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA` cipher which allows access from IE 11 on Windows Server 2008 R2 when using RSA certificates. 

Once you've run the command, you'll receive a response that looks like this:

```
ingresscontroller.operator.openshift.io/default patched
```

### Clusters using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html)

Customers who are using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html) will need to loop through each of their ingress controllers to patch each one. To patch all of your cluster's ingress controllers, run the following command:

```
for ic in $(oc get ingresscontroller -o name -n openshift-ingress-operator); do oc patch ${ic} -n openshift-ingress-operator --type=merge -p '{"spec":{"tlsSecurityProfile":{"type":"Custom","custom":{"ciphers":["TLS_AES_128_GCM_SHA256","TLS_AES_256_GCM_SHA384","ECDHE-ECDSA-AES128-GCM-SHA256","ECDHE-RSA-AES128-GCM-SHA256","ECDHE-ECDSA-AES256-GCM-SHA384","ECDHE-RSA-AES256-GCM-SHA384","ECDHE-ECDSA-CHACHA20-POLY1305","ECDHE-RSA-CHACHA20-POLY1305","DHE-RSA-AES128-GCM-SHA256","DHE-RSA-AES256-GCM-SHA384","TLS_CHACHA20_POLY1305_SHA256","TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA"],"minTLSVersion":"VersionTLS12"}}}}'; done
```

Once you've run the command, you'll receive a response that looks like this:

```
ingresscontroller.operator.openshift.io/default patched
ingresscontroller.operator.openshift.io/custom1 patched
ingresscontroller.operator.openshift.io/custom2 patched
```

## 4. Create the CronJob to ensure the TLS configuration is not overwritten

Occasionally, the cluster's ingress controller can get recreated. In these cases, the ingress controller will likely not retain the `tlsSecurityProfile` changes that we've made. To ensure this doesn't happen, we'll create a CronJob that goes through and updates the cluster's ingress controller(s). This process depends on if you are using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html).

### Clusters not using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html)

If you are not using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html), creating the CronJob is as simple as running the following command:

```
cat << EOF | oc apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: tls-patch
  namespace: openshift-ingress-operator
spec:
  schedule: '@hourly'
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: tls-patch
              image: registry.redhat.io/openshift4/ose-tools-rhel8:latest
              args:
                - /bin/sh
                - '-c'
                - oc patch ingresscontroller/default -n openshift-ingress-operator --type=merge -p '{"spec":{"tlsSecurityProfile":{"type":"Custom","custom":{"ciphers":["TLS_AES_128_GCM_SHA256","TLS_AES_256_GCM_SHA384","ECDHE-ECDSA-AES128-GCM-SHA256","ECDHE-RSA-AES128-GCM-SHA256","ECDHE-ECDSA-AES256-GCM-SHA384","ECDHE-RSA-AES256-GCM-SHA384","ECDHE-ECDSA-CHACHA20-POLY1305","ECDHE-RSA-CHACHA20-POLY1305","DHE-RSA-AES128-GCM-SHA256","DHE-RSA-AES256-GCM-SHA384","TLS_CHACHA20_POLY1305_SHA256","TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA"],"minTLSVersion":"VersionTLS12"}}}}'
          restartPolicy: Never
          serviceAccountName: cron-ingress-patch-sa
EOF
```

Note, this CronJob will run every hour, and will patch the ingress controller, if necessary. It is important that this CronJob does not run constantly, as it can trigger reconciles that could overload the OpenShift Ingress Operator. Most of the time, the logs of the CronJob pod will look something like this, as it will not be changing anything:

```
ingresscontroller.operator.openshift.io/default patched (no change)
```

### Clusters using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html)

If you are using the [Custom Domain Operator](https://docs.openshift.com/rosa/applications/deployments/osd-config-custom-domains-applications.html) the CronJob will need to loop through and patch each ingress controller. To create this CronJob, run the following command:

```
cat << EOF | oc apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: tls-patch
  namespace: openshift-ingress-operator
spec:
  schedule: '@hourly'
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: tls-patch
              image: registry.redhat.io/openshift4/ose-tools-rhel8:latest
              args:
                - /bin/sh
                - '-c'
                - for ic in $(oc get ingresscontroller -o name -n openshift-ingress-operator); do oc patch ${ic} -n openshift-ingress-operator --type=merge -p '{"spec":{"tlsSecurityProfile":{"type":"Custom","custom":{"ciphers":["TLS_AES_128_GCM_SHA256","TLS_AES_256_GCM_SHA384","ECDHE-ECDSA-AES128-GCM-SHA256","ECDHE-RSA-AES128-GCM-SHA256","ECDHE-ECDSA-AES256-GCM-SHA384","ECDHE-RSA-AES256-GCM-SHA384","ECDHE-ECDSA-CHACHA20-POLY1305","ECDHE-RSA-CHACHA20-POLY1305","DHE-RSA-AES128-GCM-SHA256","DHE-RSA-AES256-GCM-SHA384","TLS_CHACHA20_POLY1305_SHA256","TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA"],"minTLSVersion":"VersionTLS12"}}}}'; done
          restartPolicy: Never
          serviceAccountName: cron-ingress-patch-sa
EOF
```

Note, this CronJob will run every hour, and will patch the ingress controller, if necessary. It is important that this CronJob does not run constantly, as it can trigger reconciles that could overload the OpenShift Ingress Operator. Most of the time, the logs of the CronJob pod will look something like this, as it will not be changing anything:

```
ingresscontroller.operator.openshift.io/default patched (no change)
ingresscontroller.operator.openshift.io/custom1 patched (no change)
ingresscontroller.operator.openshift.io/custom2 patched (no change)
``` 
