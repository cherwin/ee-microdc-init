# Kubernetes Bootstrap

### Set up your environment

```
export CLUSTER=dev
export AWS_ACCOUNT=preprod
export PROJECT=microdc
export CIDR=10.68.0.0/16

export AWS_DEFAULT_PROFILE=${PROJECT}-${AWS_ACCOUNT}
export AWS_DEFAULT_REGION=eu-west-1
```

### Create kops state bucket

```
aws s3api create-bucket --bucket ${AWS_DEFAULT_PROFILE}-kops --region ${AWS_DEFAULT_REGION} --create-bucket-configuration=LocationConstraint=${AWS_DEFAULT_REGION}

aws s3api put-bucket-versioning --bucket ${AWS_DEFAULT_PROFILE}-kops  --versioning-configuration Status=Enabled

export KOPS_STATE_STORE=s3://${AWS_DEFAULT_PROFILE}-kops
```

### Generate a cluster name that captures both the environment and project in the name a append `.k8s.local`
```
export NAME=${CLUSTER}.${AWS_ACCOUNT}.${PROJECT}.k8s.local
```

### Allow the golang AWS SDK to assume roles
```
export AWS_SDK_LOAD_CONFIG=1
```

### Create cluster config with KOPS
```
kops create cluster --cloud aws --encrypt-etcd-storage --network-cidr ${CIDR} --authorization RBAC --topology private --networking weave --node-count 3 --zones ${AWS_DEFAULT_REGION}a,${AWS_DEFAULT_REGION}b,${AWS_DEFAULT_REGION}c --node-size m4.2xlarge --master-zones ${AWS_DEFAULT_REGION}a,${AWS_DEFAULT_REGION}b,${AWS_DEFAULT_REGION}c --master-size m4.large --kubernetes-version 1.7.10 ${NAME}
```

### Add additional cluster configuration
Add the following config using the command below, but not that the `EXTERNAL_DOMAIN` variable should be replaced with an appropriate value.
```
kops edit cluster ${NAME}
```
* Set the version of the KubeAPI to use
```
  masterPublicName: ${EXTERNAL_DOMAIN}   # this key exists so find it and change it rather then adding it in. it should be set to what ever you want the public api dns address to be, which you need to create manaually in your dns currently.
  kubeAPIServer:
    #user that's got all the rights regardless of groups, roles, what ever
    authorizationRbacSuperUser: admin
    runtimeConfig:
      #enabling a cronjob feature that's enabled by default in k8s 1.8 onwards
      batch/v2alpha1: "true"
```
* Allow k8s masters to add route53 entries and allow nodes to assume roles
```
  additionalPolicies:
    master: |
      [
        {
          "Effect": "Allow",
          "Action": [
            "route53:GetHostedZone",
            "route53:ChangeResourceRecordSets",
            "route53:ListResourceRecordSets"
          ],
          "Resource": ["arn:aws:route53:::hostedzone/*"]
        }
      ]
    node: |
      [
        {
          "Effect": "Allow",
          "Action": ["sts:AssumeRole"],
          "Resource": "*"
        }
      ]
```


### Apply the config to the cluster (if this is a first time setup this is what actually creates the cluster for the first time)
```
kops update cluster ${NAME} --yes
```

```
# we need this but it's not working: aws ec2 describe-subnets --query "Subnets[?Tags[?Key=='Name'&&"'!'"contains(Value,'utility')]&&[?Key=='KubernetesCluster'&&Value=='"$NAME"']].SubnetId" --output text
# so run this instead:
aws ec2 describe-subnets --query "Subnets[?Tags[?Key=='Name'&&contains(Value,'utility')]].SubnetId" --output text | xargs aws ec2 create-tags --tags "Key=kubernetes.io/role/internal-elb,Value=true" --resources
```

### Create a DNS Alias record on your chosen domain for the internal and external elb long names.

TODO: document an aws cli command that automates this step

### Wait for cluster to come up and apply all k8s yaml

https://github.com/EqualExpertsMicroDC/k8s-service-stack-full
