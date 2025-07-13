# EKS Ingress Lab: Path-Based Routing with AWS Load Balancer Controller

This repository contains the necessary configurations and a step-by-step guide to demonstrate path-based routing using Kubernetes Ingress on an Amazon Elastic Kubernetes Service (EKS) cluster. We will deploy two distinct web applications (`app1` and `app2`) and use the AWS Load Balancer Controller to expose them via a single Application Load Balancer (ALB), routing traffic based on URL paths.

## Table of Contents

  * [Introduction](#Introduction)
  * [Prerequisites](https://www.google.com/search?q=%23prerequisites)
  * [Architecture](https://www.google.com/search?q=%23architecture)
  * [Lab Steps](https://www.google.com/search?q=%23lab-steps)
      * [Phase 1: EKS Cluster Setup](https://www.google.com/search?q=%23phase-1-eks-cluster-setup)
      * [Phase 2: AWS Load Balancer Controller Setup](https://www.google.com/search?q=%23phase-2-aws-load-balancer-controller-setup)
      * [Phase 3: Deploy Two Sample Applications (app1 and app2)](https://www.google.com/search?q=%23phase-3-deploy-two-sample-applications-app1-and-app2)
      * [Phase 4: Configure Ingress for Path-Based Routing](https://www.google.com/search?q=%23phase-4-configure-ingress-for-path-based-routing)
      * [Phase 5: Verification and Access](https://www.google.com/search?q=%23phase-5-verification-and-access)
  * [Cleanup](https://www.google.com/search?q=%23cleanup)
  * [Troubleshooting](https://www.google.com/search?q=%23troubleshooting)
  * [Important Notes](https://www.google.com/search?q=%23important-notes)
  * [Contributing](https://www.google.com/search?q=%23contributing)
  * [License](https://www.google.com/search?q=%23license)

## Introduction

Kubernetes Ingress is an API object that manages external access to services in a cluster, typically HTTP. It provides load balancing, SSL termination, and name-based virtual hosting. In EKS, the AWS Load Balancer Controller is often used as an Ingress controller, as it provisions and manages AWS Application Load Balancers (ALBs) or Network Load Balancers (NLBs) directly from your Ingress resources.

This lab focuses on **path-based routing**, a common use case where a single ALB can direct traffic to different backend services based on the URL path (e.g., `your-alb-dns/app1` goes to `app1`, `your-alb-dns/app2` goes to `app2`).

## Prerequisites

Before you begin, ensure you have the following tools installed and configured on your local machine:

  * **AWS CLI v2:** Configured with an AWS account that has sufficient permissions to create EKS clusters, IAM roles, VPCs, and S3 buckets.
      * [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
      * [Configure AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html)
  * **kubectl:** The Kubernetes command-line tool.
      * [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
  * **eksctl:** A simple CLI tool for creating and managing EKS clusters. This greatly simplifies EKS cluster creation and management.
      * [Install eksctl]([https://www.google.com/search?q=https://eksctl.io/introduction/%23installation](https://eksctl.io/installation/))
  * **Helm v3:** The package manager for Kubernetes, used to install the AWS Load Balancer Controller.
      * [Install Helm](https://helm.sh/docs/intro/install/)

## Architecture

The setup will involve:

  * An EKS cluster with managed node groups.
  * The AWS Load Balancer Controller deployed within the cluster.
  * Two NGINX-based applications (`app1` and `app2`), each running in its own Deployment and exposed via its own ClusterIP Service.
  * A single Kubernetes Ingress resource that, via the AWS Load Balancer Controller, will provision an AWS Application Load Balancer (ALB).
  * The ALB will have listener rules configured to route requests:
      * `HTTP://<ALB_DNS_NAME>/app1` -\> `app1-service`
      * `HTTP://<ALB_DNS_NAME>/app2` -\> `app2-service`

<!-- end list -->

```
+----------------+       +-------------------+       +------------------------+
| User Browser   | ----> | AWS Application   | ----> | EKS Cluster            |
+----------------+       | Load Balancer (ALB) |     |                        |
                         +-------------------+     |   +------------------+   |
                         (Provisioned by ALB       |   | AWS Load Balancer|   |
                         Controller)               |   | Controller Pod   |   |
                                                   |   +--------+---------+   |
                                                   |            |             |
                         Path: /app1               |            | Configures  |
                         +------------------------->            | ALB Listener|
                         |                         |            | Rules/Target|
                         |                         |            | Groups      |
                         |                         |            v             |
                         |                         |   +------------------+   |
                         |                         |   | Service: app1-service|
                         |                         |   +--------+---------+   |
                         |                         |            |             |
                         |                         |   +--------v---------+   |
                         |                         |   | Deployment: app1 |   |
                         |                         |   | (NGINX Pod)      |   |
                         |                         |   +------------------+   |
                         |                         |                        |
                         | Path: /app2             |                        |
                         +------------------------->   +------------------+   |
                                                   |   | Service: app2-service|
                                                   |   +--------+---------+   |
                                                   |            |             |
                                                   |   +--------v---------+   |
                                                   |   | Deployment: app2 |   |
                                                   |   | (NGINX Pod)      |   |
                                                   |   +------------------+   |
                                                   +------------------------+
```

## Lab Steps

Follow these steps sequentially to set up and verify the Ingress.

### Phase 1: EKS Cluster Setup

1.  **Create `cluster.yaml`:**
    Create a file named `cluster.yaml` in your project directory:

    ```yaml
    # cluster.yaml
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig

    metadata:
      name: my-multi-app-eks-cluster
      region: ap-south-1 # You can change this to your preferred AWS region

    vpc:
      nat:
        gateway: Single # For cost optimization in a lab environment

    managedNodeGroups:
    - name: my-node-group
      instanceType: t3.medium # Or t2.medium for lower cost
      minSize: 2
      maxSize: 3
      desiredCapacity: 2
      volumeSize: 20
      privateNetworking: false # Keep as false for simplicity; nodes need internet access
    ```

2.  **Create the EKS Cluster:**

    ```bash
    eksctl create cluster -f cluster.yaml
    ```

    This command will take 20-30 minutes to provision the cluster and worker nodes.

3.  **Configure `kubectl`:**
    `eksctl` automatically configures your `kubeconfig`. Verify connectivity:

    ```bash
    kubectl get svc
    kubectl get nodes
    ```

### Phase 2: AWS Load Balancer Controller Setup

1.  **Enable IAM OIDC Provider:**
    This is required for IAM Roles for Service Accounts (IRSA).

    ```bash
    eksctl utils associate-iam-oidc-provider --cluster my-multi-app-eks-cluster --approve
    ```

2.  **Create IAM Policy for Controller:**
    Choose **one** of the following methods:

    **Option A: CLI Method (Recommended for automation)**

    ```bash
    curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json

    aws iam create-policy \
        --policy-name AWSLoadBalancerControllerIAMPolicy \
        --policy-document file://iam_policy.json
    ```

    Note down the `Arn` from the output (e.g., `arn:aws:iam::YOUR_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy`).

    **Option B: AWS Console Method**

      * Go to **AWS Management Console** -\> **IAM** -\> **Policies**.
      * Click **Create policy**.
      * Select the **JSON** tab and paste the content from [this raw GitHub file](https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.11.0/docs/install/iam_policy.json).
      * Click **Next**, then **Next** again.
      * For **Policy name**, enter `AWSLoadBalancerControllerIAMPolicy`.
      * Click **Create policy**.
      * Once created, copy the **ARN** of the policy.

3.  **Create IAM Service Account for Controller:**
    Replace `YOUR_ACCOUNT_ID` with your AWS account ID and `ap-south-1` with your region. Use the policy ARN obtained in the previous step.

    ```bash
    eksctl create iamserviceaccount \
        --cluster=my-multi-app-eks-cluster \
        --namespace=kube-system \
        --name=aws-load-balancer-controller \
        --attach-policy-arn=arn:aws:iam::YOUR_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
        --override-existing-serviceaccounts \
        --approve
    ```

4.  **Install AWS Load Balancer Controller using Helm:**

    Add the EKS Helm repository:

    ```bash
    helm repo add eks https://aws.github.io/eks-charts
    helm repo update
    ```

    Get your EKS cluster's VPC ID:

    ```bash
    export VPC_ID=$(aws eks describe-cluster --name my-multi-app-eks-cluster --query "cluster.resourcesVpcConfig.vpcId" --output text)
    echo "VPC ID: $VPC_ID"
    ```

    Install the controller (ensure `clusterName`, `region`, and `vpcId` are correct):

    ```bash
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
        -n kube-system \
        --set clusterName=my-multi-app-eks-cluster \
        --set serviceAccount.create=false \
        --set serviceAccount.name=aws-load-balancer-controller \
        --set region=ap-south-1 \
        --set vpcId=$VPC_ID
    ```

    Verify the deployment:

    ```bash
    kubectl get deployment -n kube-system aws-load-balancer-controller
    ```

    Wait for it to show `READY`.

5.  **Tag Subnets for ALB Auto Discovery:**
    The ALB Controller needs to know which subnets to use.

      * **Find your public subnets:**
        Go to **AWS Console** -\> **VPC** -\> **Subnets**. Filter by your EKS cluster's VPC. Identify at least two public subnets (in different Availability Zones) that your EKS worker nodes are using.

      * **Tag them (choose one method):**

        **CLI Method (replace with your actual subnet IDs):**

        ```bash
        aws ec2 create-tags \
            --resources subnet-xxxxxxxxxxxxxxxxx subnet-yyyyyyyyyyyyyyyyy \
            --tags Key=kubernetes.io/role/elb,Value=1
        ```

        **AWS Console Method:**

          * For each public subnet, select it.
          * Go to the **Tags** tab and click **Manage tags**.
          * Add a new tag: **Key**: `kubernetes.io/role/elb`, **Value**: `1`.
          * Click **Save changes**.

### Phase 3: Deploy Two Sample Applications (app1 and app2)

1.  **Create `app1.yaml`:**

    ```yaml
    # app1.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: app1-deployment
      labels:
        app: app1
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: app1
      template:
        metadata:
          labels:
            app: app1
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
            volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html
          volumes:
          - name: html-volume
            configMap:
              name: app1-html-config
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: app1-service
    spec:
      selector:
        app: app1
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: NodePort # Or ClusterIP
    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: app1-html-config
    data:
      index.html: |
        <!DOCTYPE html>
        <html>
        <head>
        <title>Application 1</title>
        <style>
          body { background-color: #f0f8ff; font-family: sans-serif; text-align: center; }
          h1 { color: #4682b4; }
        </style>
        </head>
        <body>
        <h1>Welcome to Application 1!</h1>
        <p>You reached me via the /app1 path.</p>
        </body>
        </html>
    ```

2.  **Create `app2.yaml`:**

    ```yaml
    # app2.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: app2-deployment
      labels:
        app: app2
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: app2
      template:
        metadata:
          labels:
            app: app2
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
            volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html
          volumes:
          - name: html-volume
            configMap:
              name: app2-html-config
    data:
      index.html: |
        <!DOCTYPE html>
        <html>
        <head>
        <title>Application 2</title>
        <style>
          body { background-color: #fff0f5; font-family: sans-serif; text-align: center; }
          h1 { color: #db7093; }
        </style>
        </head>
        <body>
        <h1>Welcome to Application 2!</h1>
        <p>You reached me via the /app2 path.</p>
        </body>
        </html>
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: app2-service
    spec:
      selector:
        app: app2
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: NodePort # Or ClusterIP
    ```

3.  **Apply Application Manifests:**

    ```bash
    kubectl apply -f app1.yaml
    kubectl apply -f app2.yaml
    ```

    Verify deployments and services:

    ```bash
    kubectl get deployments
    kubectl get services
    kubectl get pods
    ```

### Phase 4: Configure Ingress for Path-Based Routing

1.  **Create `path-based-ingress.yaml`:**

    ```yaml
    # path-based-ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: my-app-ingress
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
      labels:
        app: multi-app-ingress
    spec:
      rules:
      - http:
          paths:
          - path: /app1 # Requests starting with /app1 will go to app1-service
            pathType: Prefix
            backend:
              service:
                name: app1-service
                port:
                  number: 80
          - path: /app2 # Requests starting with /app2 will go to app2-service
            pathType: Prefix
            backend:
              service:
                name: app2-service
                port:
                  number: 80
    ```

2.  **Apply Ingress Resource:**

    ```bash
    kubectl apply -f path-based-ingress.yaml
    ```

### Phase 5: Verification and Access

1.  **Monitor Ingress Status:**

    ```bash
    kubectl get ingress my-app-ingress
    ```

    Wait until the `ADDRESS` field is populated with the DNS name of your ALB. This typically takes 2-5 minutes.

2.  **Test Path-Based Routing:**
    Once the ALB DNS name is available (e.g., `k8s-myappin-myappin-xxxxxxxxxx-yyyyyyyyyy.ap-south-1.elb.amazonaws.com`):

      * **Access Application 1:**
        Open your web browser and navigate to:
        `http://<ALB_DNS_NAME>/app1`
        You should see the "Welcome to Application 1\!" page.

      * **Access Application 2:**
        Open your web browser and navigate to:
        `http://<ALB_DNS_NAME>/app2`
        You should see the "Welcome to Application 2\!" page.

    Congratulations\! You have successfully demonstrated path-based routing with Ingress on EKS.

## Cleanup

**It is critical to clean up all AWS resources created during this lab to avoid incurring unnecessary costs.**

1.  **Delete Kubernetes Resources:**

    ```bash
    kubectl delete -f path-based-ingress.yaml
    kubectl delete -f app1.yaml
    kubectl delete -f app2.yaml
    ```

    Wait approximately 5-10 minutes for the ALB created by `my-app-ingress` to be de-provisioned by the AWS Load Balancer Controller. Verify its deletion in the AWS Console (EC2 -\> Load Balancers).

2.  **Delete IAM Policy:**
    Replace `YOUR_ACCOUNT_ID` with your AWS account ID.

    **If created via CLI:**

    ```bash
    aws iam delete-policy --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy
    ```

    **If created via AWS Console:**

      * Go to **AWS Console** -\> **IAM** -\> **Policies**.
      * Search for `AWSLoadBalancerControllerIAMPolicy`.
      * Select the policy and click **Delete**. Confirm.

3.  **Delete the EKS Cluster:**

    ```bash
    eksctl delete cluster --name my-multi-app-eks-cluster
    ```

    This command will delete the EKS cluster, worker nodes, associated CloudFormation stacks, and all other resources created by `eksctl`. This step also takes a significant amount of time (15-25 minutes).

## Troubleshooting

  * **Ingress `ADDRESS` not populated:**

      * Check `kubectl get events` for any errors related to Ingress or the ALB Controller.
      * Inspect the logs of the `aws-load-balancer-controller` pod:
        ```bash
        kubectl logs -n kube-system deployment/aws-load-balancer-controller
        ```
      * Verify subnet tagging in AWS Console (VPC -\> Subnets).
      * Ensure the IAM policy and service account for the controller are correctly configured and associated.
      * Check AWS CloudTrail logs for API errors related to EC2 (ALB, Target Groups, Security Groups) or IAM.

  * **Application not accessible or 404:**

      * Double-check the `path` and `service` names in your `path-based-ingress.yaml`.
      * Ensure your `Deployment` and `Service` names and labels are correct in `app1.yaml` and `app2.yaml`.
      * Verify that your `app1-deployment` and `app2-deployment` pods are `Running` and healthy (`kubectl get pods`).
      * Confirm the `targetPort` in your Service matches the `containerPort` in your Deployment.
      * Check the ALB Target Group health checks in the AWS Console.

## Important Notes

  * **Cost:** Running an EKS cluster and ALBs incurs costs. **Always perform the `Cleanup` steps immediately after completing the lab to avoid unexpected AWS charges.**
  * **IAM Roles for Service Accounts (IRSA):** This lab uses IRSA to grant the AWS Load Balancer Controller the specific AWS permissions it needs, which is a best practice for security.
  * **Target Type `ip`:** Using `alb.ingress.kubernetes.io/target-type: ip` directly registers pod IPs as targets in the ALB's target groups, which is generally more efficient than `instance` mode (which registers worker node IPs).
  * **Path Rewriting:** In this specific lab, our NGINX applications are configured to serve their content from the root `/` path within the container. The ALB forwards the *original* request path (`/app1` or `/app2`). Since our applications don't strictly care about the incoming path and just display a static page, this works. For more complex scenarios where your backend application expects a *rewritten* path (e.g., `/app1/data` on the ALB should become `/data` at the backend service), you would typically need to implement path rewriting either at the application level or via more advanced ALB Listener Rules (which are more complex to manage directly through Ingress annotations for dynamic backends).

## Contributing

Feel free to open issues or submit pull requests if you have suggestions for improvements or encounter problems.

## License

This project is open-source and available under the [MIT License](https://www.google.com/search?q=LICENSE).

-----
