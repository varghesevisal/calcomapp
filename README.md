**CALCOM APPLICATION ON EKS USING CLOUDFORMATION**

1. **Deploy EKS Stack for Calcom using the CloudFormation Template**
   
   Set the environment in your machine for the newly created cluster:

   <code>aws eks --region us-east-2 update-kubeconfig --name calcom-eks --profile pc</code>
2. **Install Add-ons: VPC CNI, Kube-Proxy, and CoreDNS**

3. **Set up the Load Balancer for the Calcom Application using Helm**
    
        eksctl utils associate-iam-oidc-provider --region=us-east-2 --cluster=calcom-eks --profile pc --approve

        curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json


        aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json --profile pc
 
        eksctl create iamserviceaccount \
        --cluster=calcom-eks \
        --namespace=kube-system \
        --name=aws-load-balancer-controller \
        --role-name AmazonEKSLoadBalancerControllerRole \
        --attach-policy-arn=arn:aws:iam::923889749700:policy/AWSLoadBalancerControllerIAMPolicy \
        --approve \
        --region us-east-2 \
        --profile pc
    

        eksctl get iamserviceaccount --cluster calcom-eks --name aws-load-balancer-controller --namespace kube-system --region us-east-2 --profile pc

        helm repo add eks https://aws.github.io/eks-charts

        kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master" 

  
        helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
        -n kube-system \
        --set clusterName=calcom-eks \
        --set serviceAccount.create=false \
        --set region=us-east-2 \
        --set vpcId=vpc-06393f6418adf94cf \
        --set serviceAccount.name=aws-load-balancer-controller 

    
4. **Steps to create a custom image for the API server**
    1. To set up API server and push the Docker image to ECR, follow these steps:

        a. Clone the Repository and Checkout Release:
            Ensure you have the repository cloned locally and navigate to it:
            Checkout the specific release tag(v3.5.0)

            git clone https://github.com/calcom/cal.com
            cd cal.com
            git checkout v3.5.0

        b. Build the Docker Image:

            docker build -t cal-api-v3.5.0 -f ./infra/docker/api/Dockerfile .
        c. Login to ECR:
           Obtain a login command for ECR and authenticate Docker to your registry:

            aws ecr get-login-password --region us-east-2 --profile pc | docker login --username AWS --password-stdin 923889749700.dkr.ecr.us-east-2.amazonaws.com
        d. Tag the Docker image with the ECR repository URI:

            docker tag cal-api-v3.5.0:latest 923889749700.dkr.ecr.us-east-2.amazonaws.com/calapi-v3.5.0:latest

        e. Finally, push the tagged Docker image to ECR:

            docker push 923889749700.dkr.ecr.us-east-2.amazonaws.com/calapi-v3.5.0:latest

5. **Deploy PostgreSQL Application using CloudFormation Stack**

   a. Copy the cluster endpoint and paste it in the <code>k-calcom-configmap.yaml</code>

6. **Deploy the Calcom Web Application**

    a. Create a Fargate profile for the Calcom namespace - It is included in the cloudformation script, we can skip this step

        eksctl create fargateprofile --cluster calcom-eks --region us-east-2 --name calcom --namespace calcom --profile pc

    b. Apply the config map that will be referenced in the deployment script. It includes namespace and config creation:
    
        kubectl apply -f k-calcom-configmap.yaml
    
    c. Deploy the Calcom application with an internet-facing Ingress ALB. The application installation takes approximately 6 to 12 minutes:
   
        kubectl apply -f k-calcom-deployment-with-alb.yaml

    d. Check the created ingress:

        kubectl get ingress/calcom-ingress -n calcom

    <code>NAME             CLASS   HOSTS   ADDRESS                                                           PORTS   AGE
    calcom-ingress   alb     *       k8s-calcom-calcomin-f5c1cdb252-1146159469.us-east-2.elb.amazonaws.com   80      8s</code>

    The load balancer address(A Record) is the url which can be used to view the ui.

    e. Verify it in the logs:

        kubectl logs deployment .apps/aws-load-balancer-controller -n kube-system

    d. Check the exposed services:

        kubectl get svc -n calcom

    <code>NAME     TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
    calcom   NodePort   10.100.133.119   <none>        80:31405/TCP   3m18s</code>

    e. List the pods: 

        kubectl get pods -o wide -n calcom

    <code>
    NAME                      READY   STATUS              RESTARTS   AGE     IP       NODE                                                   NOMINATED NODE   READINESS GATES
    calcom-575d844586-trs57   0/1     ContainerCreating   0          3m52s   <none>   fargate-ip-192-168-97-159.us-east-2.compute.internal <none>           <none> </code>

    f. Bash into the pods if required

        kubectl exec -it <podname> -n calcom -- /bin/bash
    g.  To see the created pods
    
        kubectl get pods -n calcom
        
        NAME                      READY   STATUS    RESTARTS   AGE
        calapi-5fbfb69fcf-88r6w   1/1     Running   0          41m
        calcom-575d844586-tw7wb   1/1     Running   0          41m