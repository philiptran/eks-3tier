# System Design: 3-Tier Web App on AWS EKS

This is a companion repo for the series of videos I created below. Please
note that this codebase is for educational purposes only and is NOT guaranteed
to be the best implementation for production purposes. For more technical
discussions on Kubernetes, CI/CD, and software engineering in general, please
visit [relaxdiego.com](https://relaxdiego.com).


## Video Walkthroughs

1. [Part 1](https://youtu.be/8-M2rK4NRyI)
2. [Part 2](https://youtu.be/RuCOqveUM9k)


## Prerequisites

### Set up your AWS client

First, ensure that you've configured your AWS CLI accordingly. Setting
that up is outside the scope of this guide so please go ahead and read
up at https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html


### Install Terraform

Grab the latest Terraform CLI [here](https://www.terraform.io/downloads.html)


### Install kubectl

Grab it via [this guide](https://kubernetes.io/docs/tasks/tools/#kubectl)


### Install eksctl

Grab it via [this guide](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)


### Install Helm

Grab it via [this guide](https://helm.sh/docs/intro/install/)


## Provision the Infrastructure

### Initialize the Terraform Working Directory

```
cd <PROJECT-ROOT>

terraform -chdir=terraform init
```


### Create Your Environment-Specific tfvars File

```
cp terraform/example.tfvars terraform/terraform.tfvars
```
Then modify the file as you see fit.

For EC2 key pair, generate OpenSSH key and configure in Terraform for importing into AWS instead of using AWS console to create EC2 key pair.

~~For EC2 keypair, choose PEM format and then generate the OpenSSH key format for the public key as follow.~~
~~ssh-keygen -y -f eks_jh.pem > eks_jh.pub~~
~~For PPK file, puttygen eks-jh2.ppk -O public-openssh -o eks-jh2.pub~~

### Create the DB Credentials Secret in AWS

```
secrets_dir=~/.relaxdiego/system-design/secrets
mkdir -p $secrets_dir
chmod 0700 $secrets_dir

aws_profile=$(grep -E ' *aws_profile *=' terraform/terraform.tfvars | sed -E 's/ *aws_profile *= *"(.*)"/\1/g')
aws_region=$(grep -E ' *aws_region *=' terraform/terraform.tfvars | sed -E 's/ *aws_region *= *"(.*)"/\1/g')
cluster_name=$(grep -E ' *cluster_name *=' terraform/terraform.tfvars | sed -E 's/ *cluster_name *= *"(.*)"/\1/g')
db_creds_secret_name=${cluster_name}-db-creds
db_creds_secret_file=${secrets_dir}/${cluster_name}-db-creds.json

cat > $db_creds_secret_file <<EOF
{
    "db_user": "SU_$(uuidgen | tr -d '-')",
    "db_pass": "$(uuidgen)"
}
EOF
chmod 0600 $db_creds_secret_file

aws secretsmanager create-secret \
  --profile "$aws_profile" \
  --region ${aws_region}
  --name "$db_creds_secret_name" \
  --description "DB credentials for ${cluster_name}" \
  --secret-string file://$db_creds_secret_file
```


### Create a Route 53 Zone for Your Environment

First, get a hold of an FQDN that you own and define it in an env var:

```
route53_zone_fqdn=<TYPE-IN-YOUR-FQDN-HERE>
```

Let's also create a unique caller reference:

```
route53_caller_reference=$(uuidgen | tr -d '-')
```

Then, create the zone:

```
aws_profile=$(grep -E ' *aws_profile *=' terraform/terraform.tfvars | sed -E 's/ *aws_profile *= *"(.*)"/\1/g')
aws_region=$(grep -E ' *aws_region *=' terraform/terraform.tfvars | sed -E 's/ *aws_region *= *"(.*)"/\1/g')

aws route53 create-hosted-zone \
  --profile "$aws_profile" \
  --name "$route53_zone_fqdn" \
  --caller-reference "$route53_caller_reference" > tmp/create-hosted-zone.out
```

List the nameservers for your zone:

```
cat tmp/create-hosted-zone.out | jq -r '.DelegationSet.NameServers[]'
```

Now modify your DNS servers to use the hosts listed above.


### Provision the Environment

```
terraform -chdir=terraform apply
```

Proceed once the above is done.


### (Optional) Connect to the Bastion for the First Time

Use [ssh4realz](https://github.com/relaxdiego/ssh4realz) to ensure
you connect to the bastion securely. For a guide on how to (and why)
use the script, see [this video](https://youtu.be/TcmOd4whPeQ).

```
./ssh4realz $aws_profile $(terraform -chdir=terraform output -raw bastion_instance_id)
```

TODO: How to use existing EC2 key pair and setting up the OpenSSH authorized_keys properly?

Use AWS console and EC2 Instance Connect for now...

### Subsequent Bastion SSH Connections

With the bastion's host key already saved to your known_hosts file,
just SSH directly to its public ip.

```
ssh -A ubuntu@$(terraform -chdir=terraform output -raw bastion_public_ip)
```


### Set-up Your kubectl Config File

Back in your local machine

```
aws eks --profile ${aws_profile} --region=$(terraform -chdir=terraform output -raw aws_region) \
  update-kubeconfig \
  --dry-run \
  --name $(terraform -chdir=terraform output -raw k8s_cluster_name) \
  --alias $(terraform -chdir=terraform output -raw cluster_name) | \
  sed -E "s/^( *(cluster|name)): *arn:.*$/\1: $(terraform -chdir=terraform output -raw cluster_name)/g" \
  > ~/.kube/config

kubectl config use-context $(terraform -chdir=terraform output -raw cluster_name)

chmod 0600 ~/.kube/config
```

Check that you're able to connect to the kube-api-server:

```
kubectl get pods --all-namespaces
```


### Sanity Check: Test that Pods Can Reach the DB

```
# Print out the DB endpoint for reference
terraform -chdir=terraform output db_endpoint

kubectl run -i --tty --rm debug --image=busybox --restart=Never -- sh
```

Once in the prompt, run:

```
/ # telnet <HOSTNAME-PORTION-OF-db_endpoint-OUTPUT> <PORT-PORTION-OF-db_endpoint-OUTPUT>
```

It should output:

```
Connected to <HOSTNAME>
```

To exit:

```
<Press Ctrl-] then Enter then e>
/ # exit
```


### Log in to the UI and API Container Registries

```
aws --profile $aws_profile ecr get-login-password --region $(terraform -chdir=terraform output -raw aws_region) | \
  docker login --username AWS --password-stdin $(terraform -chdir=terraform output -raw registry_frontend)

aws --profile $aws_profile ecr get-login-password --region $(terraform -chdir=terraform output -raw aws_region) | \
  docker login --username AWS --password-stdin $(terraform -chdir=terraform output -raw registry_api)
```


### Ensure Your Cluster Has an OpenID Connect Provider

OIDC will be used by some pods in the cluster to connect to the AWS API.
This section will be based off of [this guide](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)

First check if the cluster already has an OIDC provider:

```
aws --profile $aws_profile eks describe-cluster \
    --region $(terraform -chdir=terraform output -raw aws_region) \
    --name $(terraform -chdir=terraform output -raw k8s_cluster_name) \
    --query "cluster.identity.oidc.issuer" \
    --output text
```

It should return something like:

```
https://oidc.eks.us-west-2.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E
```

Now grep that sample ID from your list of OIDC providers:

```
aws --profile $aws_profile iam list-open-id-connect-providers | grep <EXAMPLED539D4633E53DE1B716D3041E>
```

If the above command returned an ARN, you're done with this section. If
it did not return one, then run:

```
eksctl --profile ${aws_profile} utils associate-iam-oidc-provider \
    --region $(terraform -chdir=terraform output -raw aws_region) \
    --cluster $(terraform -chdir=terraform output -raw k8s_cluster_name) \
    --approve
```

Rerun the aws iam command above again (including the pipe to grep) to
double check. It should return a value this time.


### Install cert-manager

While AWS has its own [certificate management service](https://aws.amazon.com/certificate-manager/),
we will work with [cert-manager](https://cert-manager.io) for this exercise
just so that we'll also learn how to use it. I mean, we're already learning
stuff, so we might as well!

```
kubectl apply --validate=false -f apps/cert-manager/cert-manager.yaml
```

Watch for the status of each cert-manager pod via:

ISSUE: cert-manager-cainjector hit error and status changed to CrashLoopBackoff

Install the latest version here directly from jetstack:

```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.9.1/cert-manager.yaml
```

```
watch -d kubectl get pods -n cert-manager
```


### Install the Load Balancer Controller

The AWS LB Controller allows us to create an AWS ALB by simply creating
Ingress resources in our k8s cluster thereby exposing our front end and
API services to the world. We will base the following steps on
[this guide](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/deploy/installation/)

```
aws --profile $aws_profile iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://apps/aws-lb-controller/iam-policy.json | \
  tee tmp/aws-load-balancer-controller-iam-policy.json

aws_account_id=$(terraform -chdir=terraform output -raw aws_account_id)
k8s_cluster_name=$(terraform -chdir=terraform output -raw k8s_cluster_name)

eksctl --profile $aws_profile create iamserviceaccount \
--cluster="$k8s_cluster_name" \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::${aws_account_id}:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve

cat apps/aws-lb-controller/load-balancer.yaml | \
  sed 's@--cluster-name=K8S_CLUSTER_NAME@'"--cluster-name=${k8s_cluster_name}"'@' | \
  kubectl apply -f -
```

FAILED: 
MountVolume.SetUp failed for volume "cert" : secret "aws-load-balancer-webhook-tls" not found

Follow the guide for EKS 1.22
https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/installation/

curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.3/docs/install/iam_policy.json

REPLACE THE OLD load-balancer.yaml with the latest one for v2.4.3 and update the placeholder for K8S_CLUSTER_NAME

Watch the aws-load-balancer-controller-xxxxxx-yyyy pod go up via:

```
watch -d kubectl get pods -n kube-system
```


### Create the cert-manager Cluster Issuer

The following steps are based off of [this guide](https://cert-manager.io/docs/configuration/acme/dns01/route53/),
and [this bit of a (working) hack](https://github.com/kubernetes-sigs/aws-load-balancer-controller/issues/1084#issuecomment-725566515):

Next, let's deploy the cluster issuer in another terminal:

```
route53_zone_fqdn=$(cat tmp/create-hosted-zone.out | jq -r '.HostedZone.Name' | rev | cut -c2- | rev)
route53_zone_id=$(cat tmp/create-hosted-zone.out | jq -r '.HostedZone.Id')
cluster_name=$(terraform -chdir=terraform output -raw cluster_name)
aws_region=$(terraform -chdir=terraform output -raw aws_region)
cert_manager_role_arn=$(terraform -chdir=terraform output -raw cert_manager_role_arn)

cat apps/cert-manager/cluster-issuer.yaml | \
  sed 's@ROUTE53_ZONE_FQDN@'"${route53_zone_fqdn}"'@' | \
  sed 's@ROUTE53_ZONE_ID@'"${route53_zone_id}"'@' | \
  sed 's@CLUSTER_NAME@'"${cluster_name}"'@' | \
  sed 's@AWS_REGION@'"${aws_region}"'@' | \
  sed 's@CERT_MANAGER_ROLE_ARN@'"${cert_manager_role_arn}"'@' | \
  kubectl apply -f -
```

Check that it created the secret for our app:

```
kubectl get secret ${cluster_name}-issuer-pkey -n cert-manager
```


### Prepare the App's Namespace

```
kubectl create ns system-design
```

### Add the DB Credentials as a Secret

NOTE: This isn't a recommended approach for production environments. If
you want something more robus, see [this guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html).

```
cluster_name=$(terraform -chdir=terraform output -raw cluster_name)

kubectl create secret generic postgres-credentials \
  -n system-design \
  --from-env-file <(jq -r "to_entries|map(\"\(.key)=\(.value|tostring)\")|.[]" ~/.relaxdiego/system-design/secrets/${cluster_name}-db-creds.json)
```


### Wait for App Events

First, lets follow events in the system-design namespace to know what's happening
when we apply our manifest later:

```
kubectl get events -n system-design -w
```


### Build and Deploy the API

```
make api
```

In the other terminal session where you're watching events, wait for this line:

```
Issuing   The certificate has been successfully issued
```

Be patient though as it can take a few minutes and you'll see errors like these:

```
Error presenting challenge: Time limit exceeded. Last error:
Failed build model due to ingress: system-design/ingress-sysem-design-api: none certificate found for host: api.<fqdn>
```

Ignore those. Check the status as well via:

```
https://check-your-website.server-daten.de/?q=${component}.${route53_zone_fqdn}
```


### Import the Key and Cert to ACM and Add the API FQDN to Route53

One downside to using cert-manager with AWS LB Controller is that they don't have
a seamless integration at the time of writing. So once cert-manager is done creating
the key and cert, we have to push them to ACM so that ALB can use them.

```
scripts/sync-tls-resources api
```

Once this script completes, the AWS LB Controller will be able to
create the ALB. Next, we update the API DNS record to point to the ALB:

```
scripts/route53-recordset CREATE api
```

Also need to edit the health checks for API Target groups as the response status code is 404 or use the correct health check target at /healthz/

Swagger endpoint: https://api.${FQDN}/docs#/

### Build and Deploy the Frontend

```
make frontend
```

In the other terminal session where you're watching events, wait for this line:

```
Issuing   The certificate has been successfully issued
```

Be patient though as it can take a few minutes and you'll see errors like these:

```
Error presenting challenge: Time limit exceeded. Last error:
Failed build model due to ingress: system-design/ingress-sysem-design-ui: none certificate found for host: ui.<fqdn>
```

Ignore those. Check the status as well via:

```
https://check-your-website.server-daten.de/?q=${component}.${route53_zone_fqdn}
```


### Import the Key and Cert to ACM and Add the Frontend FQDN to Route53

One downside to using cert-manager with AWS LB Controller is that they don't have
a seamless integration at the time of writing. So once cert-manager is done creating
the key and cert, we have to push them to ACM so that ALB can use them.

```
scripts/sync-tls-resources ui
```

Once this script completes, the AWS LB Controller will be able to
create the ALB. Next, we update the API DNS record to point to the ALB:

```
scripts/route53-recordset CREATE ui
```


### Clean Up Your Mess!

```
scripts/route53-recordset delete api
scripts/route53-recordset delete ui

kubectl delete ns system-design

scripts/delete-tls-resources api
scripts/delete-tls-resources ui

eksctl delete iamserviceaccount \
  --cluster=$(terraform -chdir=terraform output -raw k8s_cluster_name) \
  --namespace=kube-system \
  --name=aws-load-balancer-controller

terraform -chdir=terraform destroy

aws iam delete-policy --policy-arn $(cat tmp/aws-load-balancer-controller-iam-policy.json | jq -r '.Policy.Arn')
```

If you no longer plan on bringing up this cluster at a later time, clean
up the following as well:

```
aws_profile=$(grep -E ' *aws_profile *=' terraform/terraform.tfvars | sed -E 's/ *aws_profile *= *"(.*)"/\1/g')
aws_region=$(grep -E ' *aws_region *=' terraform/terraform.tfvars | sed -E 's/ *aws_region *= *"(.*)"/\1/g')
cluster_name=$(grep -E ' *cluster_name *=' terraform/terraform.tfvars | sed -E 's/ *cluster_name *= *"(.*)"/\1/g')
route53_zone_fqdn=$(cat tmp/create-hosted-zone.out | jq -r '.HostedZone.Name')
route53_caller_reference=$(uuidgen | tr -d '-')

aws secretsmanager delete-secret \
  --force-delete-without-recovery \
  --secret-id "${cluster_name}-db-creds"

aws route53 delete-hosted-zone \
  --profile "$aws_profile" \
  --name "$route53_zone_fqdn" \
  --caller-reference "$route53_caller_reference" > tmp/create-hosted-zone.out
```
