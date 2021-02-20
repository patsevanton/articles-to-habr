
sudo apt-get update && sudo apt-get install -y snapd git mc

sudo snap install microk8s --classic --channel=1.18/stable && sudo snap install helm --classic

sudo microk8s.start

sudo usermod -a -G microk8s ubuntu

sudo chown -f -R ubuntu ~/.kube

exit

sudo usermod -a -G microk8s sdpcc_deploy

sudo chown -f -R sdpcc_deploy ~/.kube

exit

alias kubectl=microk8s.kubectl

microk8s enable dns ingress storage metallb:192.168.22.7-192.168.22.7 

kubectl get all --all-namespaces

git clone https://github.com/apache/superset.git

cd superset/helm/superset

helm dependency update

sudo microk8s.kubectl config view --raw > $HOME/.kube/config

helm install --set persistence.enabled=true,service.type=LoadBalancer,ingress.enabled=true,ingress.hosts[0]=superset.192.168.22.7.xip.io  superset ./

microk8s reset --destroy-storage

