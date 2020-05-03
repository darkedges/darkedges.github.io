---
id: 39
title: "Installing cert-manager on Kubenetes with CloudFlare DNS - Update"
date: 2019-05-04T16:00:00+00:00
author: admin
layout: post
guid: http://www.darkedges.com/blog/?p=39
permalink: /2019/05/04/cert-manager-kubernetes-cloudflare-dns-update/
categories:
  - coreos
  - cert-manager
  - kubernetes
redirect_from:
  - "/2019/02/17/cert-manager-kubernetes-cloudflare-dns/"
  
---
The following is a quick start quide to deploying [cert-manager](https://docs.cert-manager.io/en/latest/getting-started/install.html#verifying-the-installation) on a single node CoreOS Kubernetes instance.

You will need to ensure that you have followed the instructions at [2019/02/17/cert-manager-failing-to-start/](2019/02/17/cert-manager-failing-to-start/) to get CoreOS Kubenetes configured correctly.

<!-- more -->

## Base Install

First you have to determine what version of `kubernetes` you have deployed and follow the appropriate guide.

### Kubernetes 1.15 or higher

```bash
kubectl create namespace cert-manager
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager.crds.yaml
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager.yaml --validate=false
```

### Kubernetes 1.14 or lower

```bash
kubectl create namespace cert-manager
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager.crds.yaml
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager-legacy.yaml --validate=false
```

You can confirm it configure correctly via the following

1. Pods are running

    ```bash
    kubectl get pods -n cert-manager
    ```

    should return similair to

    ``` bash
    NAME                                    READY     STATUS      RESTARTS   AGE
    cert-manager-688b97b9f4-n7b5q           1/1       Running     0          1m
    cert-manager-webhook-859cfcbc57-4sw42   1/1       Running     0          1m
    cert-manager-webhook-ca-sync-mhzn9      0/1       Completed   0          51s
    ````

1. Credentials installed

    ```bash
    kubectl get crd | grep cert-manager
    ```

    should return similair to

    ```bash
    certificaterequests.cert-manager.io                          2020-02-18T20:35:24Z
    certificates.cert-manager.io                                 2020-02-18T20:35:24Z
    challenges.acme.cert-manager.io                              2020-02-18T20:35:24Z
    clusterissuers.cert-manager.io                               2020-02-18T20:35:24Z
    issuers.cert-manager.io                                      2020-02-18T20:35:25Z
    orders.acme.cert-manager.io                                  2020-02-18T20:35:25Z
    ```

## Create the CloudFlare DNS issuer

1. Get your CloudFlare Global API Key from the CloudFlare Dashboard - https://dash.cloudflare.com/
   - Select Domain
   - Click `Get your API token` link
   - Click `Create Token`
   - Click `Use Template` next to `Edit Zone DNS`
   - In `Permissions` select the following
     - `Zone` - `DNS` - `Edit`
     - `Zone` - `Zone` - `Read`
   - In `Zone Resources` select `All Zones`
   - Enter a `Start Date` and `End Date`
   - Click `Contine to summary`
   - Review the details
   - Click `Create Token`
   - Click the `Copy` button to get your token and confirm it is working using the `curl` command provided on that screen, it should return similair to

      ```bash
      {
        "result": {
          "id": "e44b8bbefcf6cb9d0ddfc357a669bf12",
          "status": "active",
          "not_before": "2020-05-04T00:00:00Z",
          "expires_on": "2020-05-09T23:59:59Z"
        },
        "success": true,
        "errors": [],
        "messages": [
          {
            "code": 10002,
            "message": "This API Token can not be used before 2020-05-04T00:00:00Z",
            "type": null
          },
          {
            "code": 10003,
            "message": "This API Token was expired at 2020-05-09T23:59:59Z",
            "type": null
          }
        ]
      }
      ```

1. Issue the following to generate your CloudFlare API Key Secret

    ```bash
    API_TOKEN="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: cloudflare-api-token-secret
      namespace: cert-manager
    type: Opaque
    stringData:
      api-token: ${API_TOKEN}
    EOF
    ```

1. Create the `ClusterIssuer` configuration

    ```bash
    EMAIL_ADDRESS="xxxx@xxxx.xxx"
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: cert-manager.io/v1alpha2
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-staging
    spec:
      acme:
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        email: ${EMAIL_ADDRESS}
        privateKeySecretRef:
          name: letsencrypt-staging
        solvers:
        - dns01:
            cloudflare:
              email: ${EMAIL_ADDRESS}
              apiTokenSecretRef:
                name: cloudflare-api-token-secret
                key: api-token
    EOF
    ```

1. Create the Test Certficate

    ```bash
    DOMAIN_NAME="xxxxx.xxxxx.xxx"
    cat <<EOF | kubectl apply -f -
    ---
    apiVersion: cert-manager.io/v1alpha2
    kind: Certificate
    metadata:
      name: $(echo $DOMAIN_NAME | tr . -)
      namespace: cert-manager
    spec:
      secretName: $(echo $DOMAIN_NAME | tr . -)
      issuerRef:
        name: letsencrypt-staging
        kind: ClusterIssuer
      commonName: '${DOMAIN_NAME}'
      dnsNames:
      - ${DOMAIN_NAME}
    EOF
    ```

1. Confirm it has been created (may take a minute or 2 to generate) by issuing

    ```bash
    kubectl describe certificate $(echo $DOMAIN_NAME | tr . -) -n cert-manager
    ```

    should return similair to

    ```text
    Name:         test-darkedges-com
    Namespace:    cert-manager
    Labels:       <none>
    Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                    {"apiVersion":"cert-manager.io/v1alpha2","kind":"Certificate","metadata":{"annotations":{},"name":"test-darkedges-com","namespace":"cert-m...
    API Version:  cert-manager.io/v1alpha3
    Kind:         Certificate
    Metadata:
      Creation Timestamp:  2020-05-03T21:49:57Z
      Generation:          1
      Resource Version:    31175792
      Self Link:           /apis/cert-manager.io/v1alpha3/namespaces/cert-manager/certificates/test-darkedges-com
      UID:                 47d8629a-9a12-48d6-8b2a-4011065756d6
    Spec:
      Common Name:  test.darkedges.com
      Dns Names:
        test.darkedges.com
      Issuer Ref:
        Kind:       ClusterIssuer
        Name:       letsencrypt-staging
      Secret Name:  test-darkedges-com
    Status:
      Conditions:
        Last Transition Time:  2020-05-03T21:51:12Z
        Message:               Certificate is up to date and has not expired
        Reason:                Ready
        Status:                True
        Type:                  Ready
      Not After:               2020-08-01T20:51:11Z
    Events:
      Type    Reason     Age   From          Message
      ----    ------     ----  ----          -------
      Normal  Requested  2m3s  cert-manager  Created new CertificateRequest resource "test-darkedges-com-1505993903"
      Normal  Issued     48s   cert-manager  Certificate issued successfully
    ```

## Remove

### Remove Kubernetes 1.15 or higher

```bash

kubectl delete -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager.crds.yaml
kubectl delete -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager.yaml 
kubectl delete namespace cert-manager
```

### Remove Kubernetes 1.14 or lower

```bash
kubectl delete -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager.crds.yaml
kubectl delete -f https://github.com/jetstack/cert-manager/releases/download/v0.14.3/cert-manager-legacy.yaml
kubectl delete namespace cert-manager
```
