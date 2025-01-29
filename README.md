# Hosting DeepSeek-R1 on Amazon EKS Auto Mode

In this tutorial, we’ll walk you through how to host DeepSeek models on AWS using **Amazon EKS Auto Mode**. This approach leverages the flexibility and scalability of Kubernetes, 
while eliminating the need for you to manage the Kubernetes control plane, compute, storage, and networking components.

## Deploying DeepSeek-R1 on Amazon EKS Auto Mode

For this tutorial, we’ll use the [***DeepSeek-R1-Distill-Llama-8B***](https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Llama-8B) distilled model. 
While it requires fewer resources (like GPU) compared to the full [***DeepSeek-R1***](https://huggingface.co/deepseek-ai/DeepSeek-R1) model with 671B parameters, it provides a lighter, though less powerful, option compared to the full model. 

If you'd prefer to deploy the full DeepSeek-R1 model, simply replace the distilled model in the vLLM configuration.

### Install PreReqs

We’ll use **AWS CloudShell** for the setup in this tutorial to simplify the process.

![cloudshell.png](/static/images/cloudshell.jpg)

``` bash
# Installing kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Terraform
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform
```

### Create an Amazon  EKS Cluster w/ Auto Mode using Terraform
We'll use Terraform to easily provision the infrastructure, including a VPC, ECR repository, and an EKS cluster with Auto Mode enabled.

``` bash
# Clone the GitHub repo with the manifests
git clone https://github.com/aws-samples/deepseek-using-vllm-on-eks
cd deepseek-eks-automode

# Apply the Terraform configuration
terraform init
terraform apply

# Type 'yes' and press Enter to confirm the deployment.

# After Terraform finishes, configure kubectl with the new EKS cluster
$(terraform output configure_kubectl | jq -r)
```

### Create an EKS Auto Mode NodePool

For GPU support, we need to create a custom NodePool.

``` bash
# Create a custom NodePool with GPU support
kubectl apply -f manifests/gpu-nodepool.yaml

# Check if the NodePool is in 'Ready' state
kubectl get nodepool/gpu-nodepool
```

### Deploy  Deepseek Model


We’ll deploy the **DeepSeek-R1-Distill-Llama-8B** model using vLLM. To simplify the process, we’ve provided a sed command that allows you to easily set the model name and parameters.

``` bash
# Use the sed command to replace the placeholder with the model name and configuration parameters
sed -i '' "s|__MODEL_NAME_AND_PARAMETERS__|deepseek-ai/DeepSeek-R1-Distill-Llama-8B --max_model 2048|g" manifests/deepseek-deployment-gpu.yaml

# Deploy the DeepSeek model on Kubernetes
kubectl apply -f manifests/deepseek-deployment-gpu.yaml

# Check the pods in the 'deepseek' namespace 
kubectl get po -n deepseek
```

Initially, the pod might be in a **Pending state** while EKS Auto Mode provisions the underlying EC2 instances with the required GPU drivers.

``` bash
# Wait for the pod to reach the 'Running' state
kubectl get po -n deepseek --watch

# Verify that a new Node has been created
kubectl get nodes -l owner=data-engineer

# Check the logs to confirm that vLLM has started
kubectl logs deployment.apps/deepseek-deployment -n deepseek
```

You should see the log entry **Application startup complete** once the deployment is ready.

### Interact with the LLM

Next, we can create a local proxy to interact with the model using a curl request.

``` bash
# Set up a proxy to forward the service port to your local terminal
kubectl port-forward svc/deepseek-svc -n deepseek 8080:80 > port-forward.log 2>&1 &

# Send a curl request to the model
curl -X POST "http://localhost:8080/v1/chat/completions" -H "Content-Type: application/json" --data '{
 "model": "deepseek-ai/DeepSeek-R1-Distill-Llama-8B",
 "messages": [
 {
 "role": "user",
 "content": "What is Kubernetes?"
 }
 ]
 }'
```
The response may take a few seconds to build, depending on the complexity of the model’s output. 
You can monitor the progress via the **deepseek-deployment** logs.

### Build a Chatbot UI for the Model

While direct API requests work fine, let’s build a more user-friendly Chatbot UI to interact with the model. The source code for the UI is already available in the GitHub repository.

``` bash
# Retrieve the ECR repository URI created by Terraform
export ECR_REPO=$(terraform output ecr_repository_uri | jq -r)

# Build the container image for the Chatbot UI
docker build -t $ECR_REPO:0.1 chatbot-ui/application/.

# Login to ECR and push the image
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO
docker push $ECR_REPO:0.1

# Update the deployment manifest to use the image
sed -i "s#__IMAGE_DEEPSEEK_CHATBOT__#$ECR_REPO:0.1#g" chatbot-ui/manifests/deployment.yaml

# Generate a random password for the Chatbot UI login
sed -i '' "s|__PASSWORD__|$(openssl rand -base64 12 | tr -dc A-Za-z0-9 | head -c 16)|" chatbot-ui/manifests/deployment.yaml

# Deploy the UI and create the ingress class required for load balancers
kubectl apply -f chatbot-ui/manifests/ingress-class.yam
kubectl apply -f chatbot-ui/manifests/deployment.yaml

# Get the URL for the load balancer to access the application
echo http://$(k get ingress/deepseek-chatbot-ingress -n deepseek -o json | jq -r '.status.loadBalancer.ingress[0].hostname')
```

To access the Chatbot UI, you'll need the username and password stored in a Kubernetes secret.

``` bash
echo -e "Username=$(kubectl get secret deepseek-chatbot-secrets -n deepseek -o jsonpath='{.data.admin-username}' | base64 --decode)\nPassword=$(kubectl get secret deepseek-chatbot-secrets -n deepseek -o jsonpath='{.data.admin-password}' | base64 --decode)"
```
After logging in, you'll see a new **Chatbot tab** where you can interact with the model!

![chatbot-ui.png](/static/images/chatbot.jpg)


---
### Disclaimer

**This repository is intended for demonstration and learning purposes only.**
It is **not** intended for production use. The code provided here is for educational purposes and should not be used in a live environment without proper testing, validation, and modifications.

Use at your own risk. The authors are not responsible for any issues, damages, or losses that may result from using this code in production.
