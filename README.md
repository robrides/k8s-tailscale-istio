# k8s-tailscale-istio
Use Tailscale to access apps running on a private microk8s kubernetes cluster protected by istio
-
### Setup
#### On a host machine running Ubuntu 22.04 with virtualization enabled

```
sudo snap install multipass

curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null

curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list

sudo apt-get update && sudo apt-get install tailscale

# Start tailscale and allow ssh to the host
sudo tailscale up --ssh

# Ensure the host has enough memory and storage for all the nodes 
lsmem

# Launch cluster nodes
multipass launch --name cilium-master -m 2G -d 20G
multipass launch --name cilium-wkr1 -m 2G -d 20G
multipass launch --name cilium-wkr2 -m 2G -d 20G

#Install microk8s on the master node
multipass shell cilium-master

# On master node
sudo snap install microk8s --classic --channel=1.30/stable
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube

# Add worker nodes from master
sudo microk8s add-node
# Copy paste result to first work node; repeat command for the other node
e.g. sudo microk8s join 10.156.107.130:25000/5d6a52f02354c7feacc28c19c5c4159f/e3fe82ec4390 --worker

# Shell to each node, install microk8s and join them to the cluster
multipass shell cilium-wkr1
sudo snap install microk8s --classic --channel=1.30/stable
sudo microk8s join <from add-node on master>
exit

multipass shell cilium-wkr2
sudo snap install microk8s --classic --channel=1.30/stable --name cluster001
sudo microk8s join <from add-node on master>
exit

# Create a service on the master node so systemd will start microk8s on node startup
multipass shell cilium-master
nano /usr/local/bin/microk8s.sh

sudo chmod +x /usr/local/bin/microk8s.sh
sudo su
cat <<EOF > /etc/systemd/system/microk8s.service
[Unit]
Description=Start microk8s on startup

[Service]
ExecStart=/usr/local/bin/microk8s.sh
Type=oneshot
RemainAfterExit=no 

[Install]
WantedBy=multi-user.target
EOF

exit

# Reload the daemon and enable the service
systemctl --user daemon-reload
sudo systemctl enable microk8s.service

exit

# Validate microk8s startup is working
multipass stop cilium-master
multipass start cilium-master

# Wait a few minutes
multipass shell cilium-master
microk8s status

# Take snapshot of initial install for rollback purposes
multipass stop cilium-master
multipass snapshot cilium-master
multipass start cilium-master

# Enable community add-ons and enable cilium; this will remove calico and install cilium
microk8s enable community
microk8s enable cilium

# Exit to host machine
exit

# On the host machine, install cilium cli
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/v0.16.7/cilium-linux-amd64.tar.gz\{,.sha256sum\} 
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

```

References  
- 
- [HA Microk8s cluster with Multipass](https://ubuntu.com/tutorials/getting-started-with-kubernetes-ha#2-install-multipass)
- [HA Microk8s cluster with Miltipass blog post](https://pancho.dev/posts/multipass-microk8s-cluster/)  
- [Mutipass configs](https://multipass.run/docs/create-an-instance#heading--create-an-instance-with-custom-cpu-number-disk-and-ram)
- [Microk8s install](https://microk8s.io/docs/getting-started) 
- [Microk8s cluster](https://microk8s.io/docs/clustering)   
- [Microk8s on startup](https://askubuntu.com/questions/814/how-to-run-scripts-on-start-up) 
- [Cilium CLI install](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)
- [CNI migration strategies](https://samsungads.ca/engineering-blog/live-migrating-production-clusters-from-calico-to-cilium/)
- [Configure multipass data storage](https://multipass.run/docs/configure-multipass-storage)