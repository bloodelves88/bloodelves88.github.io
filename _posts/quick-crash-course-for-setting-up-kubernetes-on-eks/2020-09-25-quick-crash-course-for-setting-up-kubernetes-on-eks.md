---
layout: post
title: A Quick Crash Course for Setting Up Kubernetes on EKS
tags:
  - kubernetes
  - eks
  - aws
  - kubectl
---

## The Story
This past year I've been playing around with Kubernetes and EKS due to the shift to microservices in my company.

So when learning any new technology, the first thing people do is look for the documentation, basic introductions to the technology, "Getting Started" guides, and tutorials.

That worked out great!

Then I decided to dive into it. Let's set up a Kubernetes cluster on AWS's EKS. The guides were few, and if there was a guide, it was too basic. Where's the networking? Where's the namespace management? Where's the usage of sidecars? What if I wanted to put everything into a CI/CD pipeline?

There just wasn't a compiled single source of information to set up an end-to-end deployment on EKS. Everything was scattered all over the place, and I had to piece all the little bits and pieces of knowledge to come to where I am today.

I will compile these pieces of knowledge, but I will not be explaining any basics of Kubernetes. Think of this as a knowledge dump. I learn best by referencing examples so if something like this existed, I would have suffered so much less.

## Installation and Setup
I use the Linux Subsystem on Windows (Ubuntu), so do be aware of the differences if you are using another OS or distro.

### kubectl:
```bash
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```
In case the commands do not work or if you are using a different OS, refer to the guide here: [https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

### awscli:
```bash
curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"
unzip awscli-bundle.zip
sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws

# Configure the awscli, you need your access ID and access key from AWS IAM
aws configure
```
In case the commands do not work or if you are using a different OS, refer to the guides here: [https://docs.aws.amazon.com/cli/latest/userguide/install-cliv1.html](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv1.html)

### EKS Cluster
Just create a cluster from the web console here: [https://console.aws.amazon.com/eks/home](https://console.aws.amazon.com/eks/home)

Create a node group as well. A node group is a collection of EC2 or Fargate instances that your pods will be deployed on. The size of the EC2 will determine how many pods you can have deployed. Refer to this to see how many pods each instance size supports: [https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt)

### ECR Repository
Create a repository to store your Docker images here: [https://console.aws.amazon.com/ecr/home](https://console.aws.amazon.com/ecr/home)

## Docker Image Build and Upload
Replace the variables with your desired values
```bash
APP_NAME="example"
VERSION="1"
REGION="ap-southeast-1"
ECR_ROOT_URL="${AWS_ACCOUNT_NUMBER}.dkr.ecr.${REGION}.amazonaws.com"
ECR_URL="${ECR_ROOT_URL}/${APP_NAME}"

docker build -t ${APP_NAME}:${VERSION} .
aws ecr get-login-password --region ${REGION} \
  | docker login --username AWS --password-stdin ${ECR_ROOT_URL}
docker tag ${APP_NAME}:${VERSION} ${ECR_URL}:${VERSION}
docker push ${ECR_URL}:${VERSION}
```

## EKS and kubectl Setup
```bash
CLUSTER_NAME="example"
AWS_DEFAULT_REGION="ap-southeast-1"

aws eks --region "${AWS_DEFAULT_REGION}" update-kubeconfig --name "${CLUSTER_NAME}"
```

Now, you might get a `You must be logged in to the server (Unauthorized)` error when trying to use some `kubectl` commands. If you do get this error, do this:

```bash
kubectl edit -n kube-system configmap/aws-auth
```

A file editor will open. Replace the variables, and add this under the "data" section:
```yaml
mapUsers: |
  - userarn: arn:aws:iam::${AWS_ACCOUNT_NUMBER}:user/${USER}
    username: ${USER}
    groups:
      - system:masters
```

The resulting file should look something like this:

```yaml
apiVersion: v1
data:
  mapRoles: |
    - stuff
  mapUsers: |
    - userarn: arn:aws:iam::${AWS_ACCOUNT_NUMBER}:user/${USER}
      username: ${USER}
      groups:
        - system:masters
```

Save and close the file. Then run the following command:

```bash
# This USER is the IAM username of the person who wants to perform kubectl commands
USER="person"
kubectl create clusterrolebinding ops-user-cluster-admin-binding --clusterrole=cluster-admin --user=${USER}
```

## kubectl Crash Course
kubectl is something you'll be using a lot of. It lets you manage all the resources in your cluster.

First, configure your kubectl context. Generally, it's good practice to separate your environments or applications by [namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/), so just do it.

`kubectl config set-context --current --namespace=example-development`

Once this command is run, all future commands will operate under this namespace. To override this for a single command, add `--all-namespaces` (for listing resources) or `--namespace=my-namespace` (for creation, updates, or deletion).

### kubectl Command Syntax
kubectl commands are pretty well organized, following this structure:

`kubectl [action] [resource] [object name] [flags]`

Here are some examples:

```bash
kubectl create ns my-namespace
kubectl delete ns my-namespace
# lists all namespaces
kubectl get ns
# show details of a specific namespace
kubectl describe ns my-namespace

kubectl delete configmap my-config
# create will return an error if the resource is already created
# config.yaml has to be present locally in the directory you run this command at
kubectl create configmap my-config --from-file=config.yaml
# apply will create or update the resource
kubectl apply secret generic database \
  --from-literal="DATABASE_USERNAME=my-username" \
  --from-literal="DATABASE_PASSWORD=password123"

# Creates a bunch of resources that is defined in a template file
kubectl create -f ./app.yaml

# This is how you would update an existing deployment with a new Docker image
kubectl set image deployment/my-deployment my-container="docker-image-url"
```

Here are the most common resources that you would use and can apply these commands to. This list is not exhaustive and you might need to change the word to singular or plural. The entire list (with their shortnames) is [here](https://kubernetes.io/docs/reference/kubectl/overview/#resource-types).
- ns
- deployment
- pod
- secret
- configmap
- service
- cronjob
- job
- volume

## Template Files
Finally, the main part of it all.

### Deployment
This is going to include volumes, secrets, configmaps, readiness & liveness probes, and init containers. Remove what you do not need.

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: your-image-url
        ports:
        - containerPort: 3000
          protocol: TCP
        # I strongly recommend that you put these probes in,
        # so that the pod will show up with errors if there is an issue.
        # Your pod can look healthy without these probes, even if it is not.
        # These probes also restart the container if it fails.
        readinessProbe:
          tcpSocket:
            port: 3000
          initialDelaySeconds: 5 # time to wait before first probe
          periodSeconds: 10 # time to wait in between probes
        livenessProbe:
          tcpSocket:
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 20
        env:
        # secretKeyRef refers to a secret. Do this to pass secrets
        # to your container without exposing your secret in your
        # code repository or in the image itself.
        # kubectl apply secret generic database --from-literal="DATABASE_USERNAME=my-username"
        - name: DATABASE_USERNAME
          valueFrom:
            secretKeyRef:
              name: database
              key: DATABASE_USERNAME
        # CPU limits for the container. The values are in CPU units. If your node has 2 vCPUs/cores,
        # then the maximum limit you can input is 2000m or 2. Anything higher than that will
        # result in an error.
        # The request is the minimum amount of CPU that the container will reserve for itself.
        # While in operation, CPU will be between request and limit.(request < x < limit)
        # The "requests" value must be defined if you are intending to use a horizontal pod autoscaler.
        resources:
          limits:
            cpu: 100m # or 0.1
            memory: 200Mi # You can also set memory limits. Optional.
          requests:
            cpu: 50m # or 0.05
            memory: 100Mi
        # config.yaml will appear in your container in the specified mountPath
        volumeMounts:
        - name: config-volume
          mountPath: "/docker_workdir/config.yaml"
          subPath: config.yaml
      # initContainers will run before the main container. Use these
      # to do any initialization that has to run before your app
      initContainers:
      - name: my-init-container
        image: your-image-url
        # command for your container to run. This overrides the Dockerfile CMD
        args:
        - echo
        - hello
        env:
        - name: DATABASE_USERNAME
          valueFrom:
            secretKeyRef:
              name: database
              key: DATABASE_USERNAME
        volumeMounts:
        - name: config-volume
          mountPath: "/docker_workdir/config.yaml"
          subPath: config.yaml
      # Refers to a configmap created from the following command.
      # Use this to pass in configurations during the deployment process.
      # kubectl create configmap my-config --from-file=config.yaml
      volumes:
      - name: config-volume
        configMap:
          name: my-config
      # Use this to deploy this pod on a specific node.
      # Do this if you want to segregate your deployments into
      # specific nodes instead of having them all mixed up together.

      # This node selector depends on the Kubernetes labels
      # on the node group, so you need to define them on EKS.
      nodeSelector:
        Name: some-name

```

### Load Balancer
This will create a network load balancer on AWS. You'll need to do this - how else would you expose your service to the world?

```yaml
---
kind: Service
apiVersion: v1
metadata:
  name: my-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    # The following two lines will make the listener use TLS. Remove it if TCP is enough
    service.beta.kubernetes.io/aws-load-balancer-ssl-negotiation-policy: ELBSecurityPolicy-TLS-1-2-2017-01
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: your-certificate-arn
    # This will create an internal load balancer. Remove it to create an internet facing one
    service.beta.kubernetes.io/aws-load-balancer-internal: true
spec:
  ports:
  - protocol: TCP
    port: 3000
    targetPort: 3000
  selector:
    app: my-app
  type: LoadBalancer
  sessionAffinity: None
  externalTrafficPolicy: Cluster
status:
  loadBalancer: {}

```

### Node Port
If you already have a load balancer, then just use this. You shouldn't be deploying a new load balancer for each port - you'll rack up a huge bill on AWS.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80

```

### Cron Job
```yaml
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: 0 15 * * *
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: my-cronjob
            image: image-url
            args:
            - echo
            - hello
            env:
            - name: DATABASE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: database
                  key: DATABASE_USERNAME
            volumeMounts:
            - name: config-volume
              mountPath: "/docker_workdir/config.yaml"
              subPath: config.yaml
          volumes:
          - name: my-config
            configMap:
              name: my-config
          nodeSelector:
            Name: some-name
          restartPolicy: OnFailure
```

## Advanced Topics
### Inter-service Communication
If you're setting up microservices, these services will need to talk to one another. The easiest way to hack through this problem is to make your service publicly accessible and then call its public DNS endpoint. But it doesn't make sense for pods in the same cluster to communicate over the internet.

Instead, you can call the DNS names that are automatically generated when a service is created. If you have created a LoadBalancer, NodePort, or ClusterIP, you have a service.

The syntax of the DNS name is as follows:

```bash
service-name.namespace-name.svc.cluster.local
```

So, to do a HTTP GET request to another pod, you just need to do something like this:

```go
// This is assuming you exposed port 8080
resp, err := http.Get("http://my-service.my-namespace.svc.cluster.local:8080/")

// In my tests, this also worked. But putting the full address would be less error prone.
resp, err := http.Get("http://my-service.my-namespace:8080/")
```

You could also use the service's IP address, which you can obtain from `kubectl get services`.
```
NAME         TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
my-service   NodePort   172.20.123.123   <none>        8080:32417/TCP   13d
```

So, the URL you call would be `http://172.20.123.123:8080`. The problem with this is that the IP will change if you delete and recreate your service. With a DNS name, if you did not change the name, then your URLs will still work.

### Auto Scaling
I will go through two types of auto scaling. Horizontal pod scaling, and horizontal node scaling.

**Horizontal Node Scaling**

This basically scales your node group up or down, depending on the number of pods you have. It will also rearrange the pods on your nodes so as the even them out, or reduce the number of nodes used.

This is done by AWS EKS. The guide is here: [https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html)

**Horizontal Pod Scaling**

This adds and removes pods from your deployment depending on the metrics that you set. It works with CPU and memory metrics out of the box, and you can use other custom metrics.

The guide for this is here: [https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)

The horizontal pod autoscaler requires the Kubernetes Metrics Server to be installed and running, but EKS does not have this by default. You need to install it by running the following command:

```bash
# https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml

# Verify that it is running by running this command:
kubectl get deployment metrics-server -n kube-system
```

Next, you will need to set a CPU or memory request in your deployment manifest. It's seen in the Deployments section above, but here's the snippet:

```bash
# CPU limits for the container. The values are in CPU units. If your node has 2 vCPUs/cores,
# then the maximum limit you can input is 2000m or 2. Anything higher than that will
# result in an error.
# The request is the minimum amount of CPU that the container will reserve for itself.
# While in operation, CPU will be between request and limit.(request < x < limit)
# The "requests" value must be defined if you are intending to use a horizontal pod autoscaler.
resources:
  limits:
    cpu: 100m # or 0.1
    memory: 200Mi # You can also set memory limits. Optional.
  requests:
    cpu: 50m # or 0.05
    memory: 100Mi
```

Finally, create the horizontal pod autoscaler by running this command:

```bash
# The autoscaler will attempt to maintain the average CPU utilization at 50%,
# keeping the number of pods between 1 and 5 inclusive.
kubectl autoscale deployment my-deployment --cpu-percent=50 --min=1 --max=5
```

There are a lot of other things you can do with the autoscaler, but I'm just going to keep it basic. This should cover most people's needs anyway.

Some other pieces of knowledge that would be useful to know about the autoscaler:
- The default cooldown for scaling up is 0 seconds.
- The default cooldown for scaling down is 300 seconds.
- A full explanation of how the autoscaler works is here: [https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)

## Other Tricks
### Edit an existing resource
```bash
kubectl edit resource/resource_name
```

Example:
```bash
kubectl edit deployments/my-deployment
```

### Run shell in a container
```bash
kubectl exec -it pod_name -- /bin/bash
```

If you have multiple containers on a pod:
```bash
kubectl exec -it pod_name -c container_name -- /bin/bash
```

### View Logs of a Pod
```bash
kubectl logs pod_name

# If you have multiple containers in a pod
kubectl logs pod_name container_name
```

### View Your Rollout History
This will only record `kubectl set image` changes, unless you add the `--record=true` flag during `kubectl create`

```bash
kubectl rollout history deployment/my-deployment
```

### Rollback to a previous deployment
```bash
kubectl rollout undo deployment/my-deployment
kubectl rollout undo deployment/my-deployment --to-revision=xx
```

### Restart a resource
Sometimes, you just need to restart something, and this is how to do it. It will be usually used to restart a deployment, which will restart the pods. This is a rolling restart, so no worries about downtime.

Sometimes, you will need to do this to refresh the configuration files like the configmap.

```bash
kubectl rollout restart deployment my-deployment
```

### Update an existing resource
So you tweaked your deployment a little, how do you update it?

You can use apply: `kubectl apply -f ./my-deployment.yaml`, but this only works for certain resources, and you might also get a warning like this (but the command will still work):

> Warning: kubectl apply should be used on resource created by either kubectl create --save-config or kubectl apply

To get around that, do this. It basically replaces the manifest (like a kubectl edit):

```bash
kubectl create -f ./my-deployment.yaml -o yaml --dry-run=client | kubectl replace -f -
```

These commands perform a rolling update. Do ensure that you have your readiness probes set, or the older pod might get terminated before your new pod is truly ready.

## That's It!
I hope this knowledge dump helps you.
