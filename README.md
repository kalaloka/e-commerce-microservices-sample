IAM
Create a user “eks-admin” with AdministratorAccess
Create Security Credentials Access Key and Secret access key 

EC2

Create an ubuntu instance (region us-west-2)

ssh to the instance from local

Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update

Setup your access by


aws configure


Install Docker

sudo apt-get update
sudo apt install docker.io
docker ps
sudo chown $USER /var/run/docker.sock

Install kubectl
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client



Install eksctl

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

Setup EKS Cluster


eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.medium --nodes-min 2 --nodes-max 2
aws eks update-kubeconfig --region us-west-2 --name three-tier-cluster
kubectl get nodes


Run Manifests
kubectl create namespace two-tier-ns
kubectl apply -f .
Kubectl delete -f .


eksctl delete cluster --name my-cluster --region us-west-2


Install AWS Load Balancer

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

aws iam create-policy     --policy-name AWSLoadBalancerControllerIAMPolicy     --policy-document file://iam_policy.json

eksctl utils associate-iam-oidc-provider --region=us-west-2 --cluster=my-cluster --approve

eksctl create iamserviceaccount   --cluster=my-cluster   --namespace=kube-system   --name=aws-load-balancer-controller   --role-name AmazonEKSLoadBalancerControllerRole   --attach-policy-arn=arn:aws:iam::626072240565:policy/AWSLoadBalancerControllerIAMPolicy --approve --region=us-west-2



sudo snap install helm --classic

helm repo add eks https://aws.github.io/eks-charts

helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller   -n kube-system   --set clusterName=my-cluster   --set serviceAccount.create=false   --set serviceAccount.name=aws-load-balancer-controller

kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl apply -f full_stack_lb.yaml


Deploying One micro service which is products-cna-microservice

Dockerfile
# Use Node.js 18 on Alpine Linux
FROM node:18-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy package.json to the working directory
COPY package.json .

# Install dependencies
RUN npm install

# Copy the rest of the application code
COPY . .

# Expose the port the app runs on (e.g., 3000 if that's your app's port)
EXPOSE 3000

# Command to run the application
CMD ["npm", "start"]

docker build -t your-image-name .
Create an ECR Repository
aws ecr create-repository --repository-name your-repository-name --region your-region
aws ecr get-login-password --region your-region | docker login --username AWS --password-stdin your-account-id.dkr.ecr.your-region.amazonaws.com
docker tag products-cna-microservice:latest your-account-id.dkr.ecr.your-region.amazonaws.com/products-cna-microservice:latest
docker push your-account-id.dkr.ecr.your-region.amazonaws.com/products-cna-microservice:latest

Create Kubernetes Manifests
Deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: products-cna-microservice
spec:
  replicas: 3 # For high availability
  selector:
    matchLabels:
      app: products-cna-microservice
  template:
    metadata:
      labels:
        app: products-cna-microservice
    spec:
      containers:
      - name: products-cna-microservice
        image: your-account-id.dkr.ecr.your-region.amazonaws.com/products-cna-microservice:latest
        ports:
        - containerPort: 3000
        env:
        - name: MONGO_URI
          valueFrom:
            secretKeyRef:
              name: mongo-secrets
              key: mongo-uri
        - name: DATABASE
          valueFrom:
            configMapKeyRef:
              name: products-configmap
              key: database

service.yml
apiVersion: v1
kind: Service
metadata:
  name: products-cna-service
spec:
  type: LoadBalancer
  selector:
    app: products-cna-microservice
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000

Ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: products-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - your-domain.com
    secretName: products-cna-tls
  rules:
  - host: your-domain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: products-cna-service
            port:
              number: 80
        
Secret & ConfigMap:
secret
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secrets
type: Opaque
data:
  mongo-uri: base64-encoded-mongo-uri

CongigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: products-configmap
data:
  database: e-commerc

kubectl apply -f secret.yaml
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml

Persistent Volume & Persistent Volume Claim:
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: "/data/mongo"

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard

Set Up SSL with Cert-Manager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.7.1/cert-manager.yaml

apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx


    













