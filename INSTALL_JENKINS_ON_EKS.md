#### Install Kubectl ####
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
kubectl version --client

#### Install Helm ####
wget https://github.com/helm/helm/releases/download/v3.12.2/helm-v3.12.2-darwin-amd64.tar.gz.asc
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm

#### check version ####
kubectl version --short --client
eksctl version

#### Create EKS Cluster #####
eksctl create cluster --name cbjenkins --region ap-northeast-1 --zones ap-northeast-1a,ap-northeast-1c,ap-northeast-1d --managed --nodegroup-name jenkinsnodegroup
kubectl get svc

#### Create Security Group #####
aws ec2 create-security-group --region ap-northeast-1 --group-name efs-mount-sg --description "Amazon EFS for EKS, SG for mount target" --vpc-id vpc-0506597cbb8512f96
aws ec2 authorize-security-group-ingress --group-id sg-018a82e081e1fd6d5 --region ap-northeast-1 --protocol tcp --port 2049 --cidr 192.168.0.0/16
aws efs create-file-system --creation-token creation-token --performance-mode generalPurpose --throughput-mode bursting --region ap-northeast-1 --tags Key=Name,Value=cbjenkins--encrypted
aws ec2 describe-instances --filters Name=vpc-id,Values=vpc-0506597cbb8512f96
aws efs create-mount-target --file-system-id fs-071ceab085eeb55a6 --subnet-id subnet-01b6aabcaabd7852b --security-group sg-018a82e081e1fd6d5 --region ap-northeast-1
aws efs create-access-point --file-system-id fs-071ceab085eeb55a6 --posix-user Uid=1000,Gid=1000 --root-directory="Path=/jenkins,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=777}"

#### Deploy Jenkins ####
kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=master"
vi storageclass.yaml
vim persistentvolume.yaml
vim persistentvolumeclaim.yaml
kubectl apply -f storageclass.yaml,persistentvolume.yaml,persistentvolumeclaim.yaml
kubectl get sc,pv,pvc
helm install jenkins stable/jenkins --set rbac.create=true,master.servicePort=80,master.serviceType=LoadBalancer,persistence.existingClaim=efs-claim
printf $(kubectl get secret --namespace default jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
export SERVICE_IP=$(kubectl get svc --namespace default jenkins --template "{{ range (index .status.loadBalancer.ingress 0) }}{{ . }}{{ end }}")
printf $(kubectl get service jenkins -o jsonpath="{.status.loadBalancer.ingress[].hostname}");echo
printf $(kubectl get secret jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
kubectl get pods
kubectl exec -it jenkins-78b5d7c567-tm6nz /bin/sh
