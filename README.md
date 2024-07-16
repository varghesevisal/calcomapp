**CALCOM APPLICATION ON EKS USING CLOUDFORMATION**

1. **Deploy EKS Stack for Calcom using the CloudFormation Template**
   
   Set the environment in your machine for the newly created cluster:

   <code>aws eks --region us-east-2 update-kubeconfig --name calcom-eks --profile pc</code>
2. **Install Add-ons: VPC CNI, Kube-Proxy, and CoreDNS**

3. **Set up the Load Balancer for the Calcom Application using Helm**
    
   <code>eksctl utils associate-iam-oidc-provider --region=us-east-2 --cluster=calcom-eks --profile pc --approve</code>

   <code>curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json</code>


   <code>aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json --profile pc </code>
 
    <code>eksctl create iamserviceaccount \
        --cluster=calcom-eks \
        --namespace=kube-system \
        --name=aws-load-balancer-controller \
        --role-name AmazonEKSLoadBalancerControllerRole \
        --attach-policy-arn=arn:aws:iam::923889749700:policy/AWSLoadBalancerControllerIAMPolicy \
        --approve \
        --region us-east-2 \
        --profile pc
    </code>

    <code>eksctl get iamserviceaccount --cluster calcom-eks --name aws-load-balancer-controller --namespace kube-system --region us-east-2 --profile pc
    </code>

    <code>helm repo add eks https://aws.github.io/eks-charts</code>

    <code>kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller/crds?ref=master" </code>

    <code>
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    -n kube-system \
    --set clusterName=calcom-eks \
    --set serviceAccount.create=false \
    --set region=us-east-2 \
    --set vpcId=vpc-0256a3a9500e739a5 \
    --set serviceAccount.name=aws-load-balancer-controller 
    </code>
    

4. **Deploy PostgreSQL Application using CloudFormation Stack**

   a. Copy the cluster endpoint and paste it in the <code>k-calcom-configmap.yaml</code>

5. **Deploy the Calcom Web Application**

    a. Create a Fargate profile for the Calcom namespace - It is included in the cloudformation script, we can skip this step

    <code>eksctl create fargateprofile --cluster calcom-eks --region us-east-2 --name calcom --namespace calcom --profile pc</code>

    b. Apply the config map that will be referenced in the deployment script. It includes namespace and config creation:
    
    <code>kubectl apply -f k-calcom-configmap.yaml</code>
    
    c. Deploy the Calcom application with an internet-facing Ingress ALB. The application installation takes approximately 6 to 12 minutes:
   
    <code>kubectl apply -f k-calcom-deployment-with-alb.yaml</code>

    d. Check the created ingress:

    <code>kubectl get ingress/calcom-ingress -n calcom</code>

    <code>NAME             CLASS   HOSTS   ADDRESS                                                           PORTS   AGE
    calcom-ingress   alb     *       k8s-calcom-calcomin-f5c1cdb252-1146159469.us-east-2.elb.amazonaws.com   80      8s</code>

    e. Verify it in the logs:
    <code>kubectl logs deployment
    .apps/aws-load-balancer-controller -n kube-system</code>

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
