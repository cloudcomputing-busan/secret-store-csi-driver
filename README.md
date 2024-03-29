# What is Secret Store CSI Driver
## Overview
Kubernetes secrets are stored with Base64 encoding. etcd provides encryption at rest for these secrets, but when secrets are retrieved, they are decrypted and presented to the user. If role-based access control is not configured properly on your cluster, anyone with API or etcd access can retrieve or modify a secret. Additionally, anyone who is authorized to create a pod in a namespace can use that access to read any secret in that namespace.

To store and manage your secrets securely, you can configure the OpenShift Container Platform Secrets Store Container Storage Interface (CSI) Driver Operator to mount secrets from an external secret management system, such as Azure Key Vault, by using a provider plugin. Applications can then use the secret, but the secret does not persist on the system after the application pod is destroyed.

The Secrets Store CSI Driver Operator, ```secrets-store.csi.k8s.io```, enables OpenShift Container Platform to mount multiple secrets, keys, and certificates stored in enterprise-grade external secrets stores into pods as a volume. The Secrets Store CSI Driver Operator communicates with the provider using gRPC to fetch the mount contents from the specified external secrets store. After the volume is attached, the data in it is mounted into the container’s file system. Secrets store volumes are mounted in-line.

## Secrets store providers
The following secrets store providers are available for use with the Secrets Store CSI Driver Operator:

- AWS Secrets Manager
- AWS Systems Manager Parameter Store
- Azure Key Vault

## About CSI
Storage vendors have traditionally provided storage drivers as part of Kubernetes. With the implementation of the Container Storage Interface (CSI), third-party providers can instead deliver storage plugins using a standard interface without ever having to change the core Kubernetes code.

CSI Operators give OpenShift Container Platform users storage options, such as volume snapshots, that are not possible with in-tree volume plugins.

## Secret Store CSI Driver 설치

#### helm install
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

#### Deployment using Helm
##### Secrets Store CSI Driver allows users to customize their installation via Helm.
``` Recommended to use Helm3 ```
```
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system
```
Running the above ```helm install``` command will install the Secrets Store CSI Driver on Linux nodes in the ```kube-system``` namespace.

**배포 확인**

```kubectl get crd -n kube-system```

<img width="991" alt="image" src="https://github.com/cloudcomputing-busan/secret-store-csi-driver/assets/86287920/809439d1-4317-492f-a5a2-736da6816625">

## AWS Provider 설치
```

kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml

```

## AWS 자격증명 설정
```
CLUSTERNAME=<CLUSTER 이름>
REGION=ap-northeast-2
POLICY_ARN=$(aws --region "$REGION" --query Policy.Arn --output text iam create-policy --policy-name csi-eks-access-secrets-manager --policy-document '{
    "Version": "2012-10-17",
    "Statement": [ {
        "Effect": "Allow",
        "Action": [
          "secretsmanager:GetSecretValue",
          "secretsmanager:DescribeSecret"
        ],
        "Resource": [
          "arn:aws:secretsmanager:*:*:secret:*"
        ]
    } ]
}')
eksctl create iamserviceaccount --name csi-secrets-manager-sa --region="$REGION" --cluster "$CLUSTERNAME" --attach-policy-arn "$POLICY_ARN" --approve --override-existing-serviceaccounts
```

## Secret Provider

``` secretProvider.yml```

```
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: db-secrets
spec:
  provider: aws
  parameters:
    objects: |
        - objectName: "<SecretManager 이름>"
          objectType: "secretsmanager"
```

``` kubectl apply -f secretProvider.yml```

## Deployment with CSI Secret Manager

``` deployment.yml ```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: csi-secrets-manager-sa
      volumes:
        - name: db-creds
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: db-secrets
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
          - name: db-creds
            mountPath: /tmp
```
