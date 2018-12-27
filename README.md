# KodeDeploy

Continuous deployment of your application on Kubernetes. It is continuous that all the Kubernetes applications will be automatically redeployed, whenever you create new Kubernetes clusters.

## Rationale

As an infrastructure engineer and/or a ClusterOps person, whenever I create a new cluster, I want my cluster to deploy everything needed automatically, without requiring me to run a bunch of commands.

## What we get from KodeDeploy

All you have to in order to setup the new Kubernetes cluster is now installing an agent per a Kubernetes namespace.

AWS CodeDeploy and the KodeDeploy agent, derived from codedeploy-agent, takes care of the rest, fetching the desired state of the cluster, automatically deploying everything, for you.

This is especially helpful in the two use-cases below:

- Delegate continuous (re)deployment of apps to your product teams
- Easy Kubernetes cluster migration

### Delegate continuous (re)deployment of apps to your product teams

You don't need to remember which namespace contains which Kubernetes resources!

Aa a ClusterOps person, whenever a new project starts, you just create a dedicated Kubernetes namespace, installing a CodeDeploy agent. Finally give IAM role accesses for all the developers in the project team, and you're done.

Any developers in the project team, as long as they're allowed by AWS IAM policies, can deploy your Kubernetes applications with CodeDeploy from anywhere.

### Easy Kubernetes cluster migration

KodeDeploy allows you to easily to a kind of canary deployments of Kubernetes clusters. Run multiple flavors of Kubernetes clusters for smooth migration without headache of (re)deploying apps.

Usually doing things like this requires you to repeat `kubectl` or `helm` or something similar per Kubernetes cluster when you have multiple of them.

With KodeDeploy, all you need is installing CodeDeploy agents. Each agent knows which revision of your apps should be installed in a namespace, so that it can deploy it for you.

One standard use-case of KodeDeploy is easing Kubernetes version upgrades in your production environment. Vary the version number of Kubernetes across clusters, like `v1.10.7` or `v.1.11.3`, connecting to the same Target Group, tweaking sizes Auto Scaling Groups of your Kubernetes nodes across clustesr to for load-balancing across clusters.

## Getting started

Create a config file for the `kode` command:

```
$ AWS_DEFAULT_REGION=us-west-2
$ echo <EOF > kode.yaml
bucket: my-codedeploy-app-bundles-us-west-2
application: lb-app-1
environment: sandbox
EOF
```

Provision a set of AWS resources for kodedeploy:

```
$ ./kode plan
$ ./kode apply
```

Or provision them manually:

```
# S3 bucket for CodeDeploy
# aws s3 create-bucket --bucket-name mybucket

# CodeDeploy service role
$ aws iam create-role --role-name codedeploy-service --assume-role-policy-document file://$(pwd)/codedeploy-service.assumerolepolicy.json
$ aws iam attach-role-policy --role-name codedeploy-service --policy-arn arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole

# IAM policy for namespace agents
$ aws iam create-policy --policy-name codedeploy-namespace --policy-document file://$(pwd)/codedeploy-namespace.iampolicy.json

# IAM policy for node agents
$ aws iam create-policy --policy-name codedeploy-node --policy-document file://$(pwd)/codedeploy-node.iampolicy.json
```

To start managing apps in the clusters:

```
$ go get -u github.com/mumoshu/variant

$ git clone https://github.com/mumoshu/kodedeploy

# For each namespace in the all the participated clusters, run:

$ ./kode init --cluster cluster1 --environment staging --namespace mycoolproduct

$ ns_policy_arn=$(aws iam list-policies | jq -r '.Policies[] | select(.PolicyName == "codedeploy-namespace") | .Arn')
$ aws iam attach-user-policy --user-name name_of_the_registered_onprem_instance --policy-arn $ns_policy_arn

# To deploy an application only to a single cluster "cluster1", run:

$ ./kode deploy --cluster cluster1  --environment staging --namespace mycoolproduct --application mymicrosvc --command "kubectl apply -n mycoolproduct apply -f deploy/kubernetes" --bucket "my-bucket-for-app-deployment-bundles"

# To deploy an application to all the clusters in the environment `foo`, run:

$ ./kode deploy --environment staging --namespace mycoolproduct --application mycoolproduct --command "kubectl apply -n mycoolproduct apply -f deploy/kubernetes" --bucket "my-bucket-for-app-deployment-bundles"
```

To start managing node in the clusters:

```
$ go get -u github.com/mumoshu/variant

$ git clone https://github.com/mumoshu/kodedeploy

$ aws iam attach-role-policy --role-name eksctl-attractive-sculpture-15457-NodeInstanceRole-1J1JD1JTQZ16Q --policy-arn arn:aws:iam::AWS_ACCOUNT_ID:policy/codedeploy-node

$ kubectl apply -f codedeploy-node.daemonset.yaml
```

Then, create an ALB/NLB and a target group for your app, and a deployment group that includes all the nodes in the first cluster that runs the codedeploy-node pods.

A blue-green deployment can be triggered by creating a deployment that specifies an another ASG that contains all the nodes in the second cluster that runs the codedeploy-node pods.

```
# The CodeDeploy application to which start sending traffic
$ app=lb-app-1
$ blue_cluster_name=$(eksctl get cluster | grep blue | awk '{ print $1 }')
$ green_cluster_name=$(eksctl get cluster | grep green | awk '{ print $1 }')
$ ./kode release cluster --application ${app} --cluster ${blue_cluster} --bucket mybucket
$ ./kode release cluster --application ${app} --cluster ${green_cluster} --bucket mybucket
```

## How it works

If you're familiar with AWS, I'd say that it works much like Launch Configuration, for Kubernetes clusters.

Your launch configuration might have been automatically creating identical EC2 instances for you according to `userdata`. KodeDeploy, on the other hand, redeploys every k8s app needed for your k8s cluster.

With KodeDeploy, all you need to do for recreating your Kubernetes cluster becomes just two steps. The first step is provisioning a "raw" cluster with a tool like eksctl, kops, or kube-aws. The second and the final step is deploying `KodeDeploy`, providing `Environment Name` like `production` and `Namespace` like `my-accounting-product` or `my-analytics-platform`, onto your cluster. KodeDeploy remembers which k8s apps is needed for the cluster in e.g. the `my-analytics-platform` namespace within the `production` environment, so that it can deploy everything for you.

KodeDeploy, as you might have guessed from its name, exploits AWS CodeDeploy for Kubernetes. If you know much about CodeDeploy, the core idea of KodeDeploy is treat each CodeDeploy on-premise machine as one Kubernetes namespace in a cluster. So that a CodeDeploy deployment, formerly considered as a series of commands that is uniformely run against each machine in a deployment group, is now run against each targeted namespace in all the clusters in a targeted environment. That is, it's like KodeDeploy running your deployment in all the `myplatform` namespaces in all the `production` clusters. This is done no matter how many clusters you have in the `production` environment, hence you are freed from repeating deployments for all the clusters.

## Implementation

KodeDeploy is just a set of helper scripts and a Kubernetes manifest to deploy CodeDeploy Agent along with a Docker daemon as a sidecar.

## KodeDeploy The Hard Way

### Setup the runtime environment for agents

You should create a Kubernetes namespace and a deployment for `kodedeploy` agent. Each pod should have an IAM role to make codedeploy-agent to work properly.

Assming you have `kiam` installed in the `kube-system` namespace, the following steps result in creating the `kodedeploy` namespace with all the IAM roles permitted.

It also creates a Kubernetes deployment that creates a `kodedeploy-agent` pod that requests an IAM role named `kodedeploy-agent`. You should give CodeDeploy-related IAM permissions to the role before you proceed with actual testing.

```console
$ kubectl apply -f namespace.yaml
$ kubectl apply -f deploy.yaml
```

### Installing agents

```console
$ aws deploy register \
  --instance-name ${instance_name} \
  --tags Key=kodedeployenv,Value=production Key=kodedeploycluster,Value=examplecom Key=kodedeployns,Value=my-analytics-platform \
  --region us-west-2
```

Expect log messages like the below if you've successfully registered the instance:

```console
Creating the IAM user... DONE
IamUserArn: arn:aws:iam::YOUR_AWS_ACCOUNT:user/AWS/CodeDeploy/YOUR_INSTANCE_NAME
Creating the IAM user access key... DONE
AccessKeyId: YOUR_AUT_GENERATED_KEY_ID
SecretAccessKey: YOUR_AUTO_GENERATED_SECRET_KEY
Creating the IAM user policy... DONE
PolicyName: codedeploy-agent
PolicyDocument: {
    "Version": "2012-10-17",
    "Statement": [ {
        "Action": [ "s3:Get*", "s3:List*" ],
        "Effect": "Allow",
        "Resource": "*"
    } ]
}
Creating the on-premises instance configuration file named codedeploy.onpremises.yml...DONE
Registering the on-premises instance... DONE
Adding tags to the on-premises instance... DONE
Copy the on-premises configuration file named codedeploy.onpremises.yml to the on-premises instance, and run the following command on the on-premises instance to install and configure the AWS CodeDeploy Agent:
aws deploy install --config-file codedeploy.onpremises.yml
```

The `instance_name` should be `${env}-${cluster}-${ns}` whereas each component is:

- `kodedeployenv`: the name of the environment your cluster is in e.g. `production`, `staging`, `preview`
- `kodedeploycluster`: the name of the cluster like `examplecom`, `internalonly`
- `kodedeployns`: the namespace like `my-accounting-product`, `my-analytics-platform`

The `aws deploy register` command produces `codedeploy.onpremises.yml` that looks like:

```
---
region: ap-northeast-1
iam_user_arn: arn:aws:iam::YOUR_AWS_ACCOUNT:user/AWS/CodeDeploy/YOUR_INSTANCE_NAME
aws_access_key_id: YOUR_AUTO_GENERATED_KEY_ID
aws_secret_access_key: YOUR_AUTO_GENERATED_SECRET_KEY
```

Place this into `somedir/codedeploy.onpremises.yml` and run:

```console
kubectl --namespace ${kodedeployns} \
  create secret generic --from-file=somedir etc-codedeploy-agent-conf
```

Create a `anotherdir/codedeployagent.yml` that looks like:

```yaml
---
:log_aws_wire: false
:log_dir: '/var/log/aws/codedeploy-agent/'
:pid_dir: '/opt/codedeploy-agent/state/.pid/'
:program_name: codedeploy-agent
:root_dir: '/opt/codedeploy-agent/deployment-root'
:verbose: false
:wait_between_runs: 1
:proxy_uri:
:max_revisions: 5
```

and run:

```console
kubectl --namespace ${kodedeployns} \
  create configmap --from-file=anotherdir opt-codedeploy-agent-conf
```

finally install the codedeploy agent onto your ns by running:

```console
kubectl --namespace ${kodedeployns} \
  create -f deploy.yaml
```

And repeat these steps for each `kodedeployns` within your cluster in the environment.

### Author `appspec.yml`

`appspec.yml` is a kind of your deployment script that is run by CodeDeploy agents.

Being locked-in to Kubernetes for this specific use-case, we have less things to do here compared to traditinoal `appspec.yml` files.

Suppose you have all the files required to deploy your app onto Kubernetes under `./myapp`, always start with a `appspec.yml` like the below:

```yaml
version: 0.0
os: linux
files:
  - source: deploy
    destination: /deploy
hooks:
  AfterInstall:
  - location: codedeploy/after-install.sh
    timeout: 180
```

Whereas the `after-install.sh` looks like:

```yaml
#!/usr/bin/env bash

set -vx

wd="/opt/codedeploy-agent/deployment-root/${DEPLOYMENT_GROUP_ID}/${DEPLOYMENT_ID}/deployment-archive"
image="quay.io/roboll/helmfile:v0.40.1"

# To use kodedeploy pod's serviceaccount to access Kubernetes API
sd="/var/run/secrets/kubernetes.io/serviceaccount/"

cmd="helmfile apply"

docker run -v "${wd}:${wd}" -v "${sd}:${sd}" \
    -e "KUBERNETES_SERVICE_HOST=${KUBERNETES_SERVICE_HOST}" \
    -e "KUBERNETES_SERVICE_PORT=${KUBERNETES_SERVICE_PORT}" \
    -e "KUBE_DNS_SERVICE_HOST=${KUBE_DNS_SERVICE_HOST}" \
    -e "KUBE_DNS_SERVICE_PORT=${KUBE_DNS_SERVICE_PORT}" \
    --rm -w "${wd}" \
    "${image}" bash -c "${cmd}"
```

Edit the above `appspec.yml` to use whatever `image` and `cmd` you like, so that any tool that speaks to Kubernetes can be integrated with AWS CodeDeploy.

In case you'd want to understand how this works, `DEPLOYMENT_GROUP_ID` is a variable provided by CodeDeploy, containing the ID of the deployment group in which the current deployment is.

### Deploying revisions

Firstly, create an CodeDeploy application that hosts that is required to associate deployment groups and deployment to:

```console
aws deploy create-application --application-name ${app}
```

Run the following command to create a revision from your local source, and then deploy it:

```console
aws deploy push \
  --application-name ${app} \
  --description "${myrev}" \
  --ignore-hidden-files \
  --s3-location s3://${bucket}/${key} \
  --source ${source}
```

```console
aws deploy create-deployment \
  --application-name ${app} \
  --deployment-config-name CodeDeployDefault.OneAtATime \
  --deployment-group-name ${group} \
  --s3-location bucket=${bucket},key=${key},bundleType=${bundletype}
```

I'd suggest the following naming conventions for the variables:

`app`: the name of one microservice within your namespace. Note that each namespace could contain one more more microservices.

`env`: the name of the environnment which the agent is intended to manage e.g. `production`, `staging`, `test`, `preview`.

`ns`: the name of the Kubernetes namespace which the agent is intended to manage, recommended to be either your team's name or product name.

`group`: `${env}-${ns}` and `${env}-${ns}-${cluster}`. Create a deployment group for each combination. Use the former for continuous (re)deployment, the latter for one-shot jobs.

Now Wait for a few seconds to see the agent deploys your Kubernetes resources.

The codedeploy agent in your namespace detects the newly created AWS CodeDeploy `revision`, runs commands provided in the AWS CodeDeploy `appspec.yml` included in the source.

Now that the latest `revision` is memorized by AWS CodeDeploy, every newly installed agent automatically fetches the latest revision for installing.

See `example/push` to see how you could automate the most of these steps and conventions for you.

## Integrations

### Helmfile

Have one desired-state file for your whole namespace, that is applied automatically

- Write your desired state file called `helmfile.yaml` per namespace using [roboll/helmfile](https://github.com/roboll/helmfile)
- Modify your AWS CodeDeploy `appspec.yml` to call the `helmfile apply` command

### GitHub

See the progresses of your deplyoments in GitHub pull requests.

- Create a GitHub Webhook endopoint, with [aws-lambda-go-api-proxy](https://github.com/awslabs/aws-lambda-go-api-proxy), that reacts to `GitHub Deployment` events by creating corresponding CodeDeploy deployments.

- Use [remind101/deploy](https://github.com/remind101/deploy) to trigger `GitHub Deployment`s

### Slack

Trigger deployments via Slack.

- Use [remind101/slashdeploy](https://github.com/remind101/slashdeploy) so that you can say `/deploy` in your Slack channel to trigger `GitHub Deployment`s, which then triggers CodeDeploy deployments via the lambda function created in the above `GitHub` section
