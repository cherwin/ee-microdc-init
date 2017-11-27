# Kubernetes Bootstrap

### Create kops state bucket

```
aws s3api create-bucket --bucket microdc-kops-preprod --region eu-west-1
aws s3api put-bucket-versioning --bucket microdc-kops-preprod  --versioning-configuration Status=Enabled

KOPS_STATE_STORE=s3://microdc-kops-preprod
```

### Create a folder for k8s cluster and export as a variable
```
mkdir alan.preprod.microdc.k8s.local
cd alan.preprod.microdc.k8s.local
export NAME=$(basename $(pwd))
```

### Allow Creation of AWS Session With Option Overrides
```
export AWS_SDK_LOAD_CONFIG=1
```

### Create cluster config with KOPS
```
kops create cluster --topology private --networking weave --node-count 3 --zones eu-west-1a,eu-west-1b,eu-west-1c --node-size m4.2xlarge --master-zones eu-west-1a,eu-west-1b,eu-west-1c --master-size m4.large --kubernetes-version 1.7.9 ${NAME}
```

### Add additional cluster configuration
Add the following config using the command below
```
kops edit cluster ${NAME}
```
* Set the version of the KubeAPI to use
```
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
aws ec2 describe-subnets --query "Subnets[?Tags[?Key=='Name'&&"'!'"contains(Value,'utility')]].SubnetId" --output text | xargs aws ec2 create-tags --tags "Key=kubernetes.io/role/internal-elb,Value=true" --resources
```

### Save the cluster to a file
```
kops get cluster $NAME -o yaml --full > kops.yaml
```
### Delete and restore a cluser from a file
```
kops delete cluster --name $NAME --yes
kops create -f kops.yaml
kops update cluster ${NAME} --yes
```
