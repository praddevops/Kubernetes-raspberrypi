## Setting up Kubernetes cluster on Raspberry pi 4

1. Install minimal Raspbian for Raspberry Pi 4 from any of the mirrors and login
2. Connect to WIFI
    * If network manager is not configured on Raspbian, enter the command `sudo raspi-config` and go to Advanced Options >> Network Config and select Network Manager as the Network configuration tool
    * Using nmcli (https://nullr0ute.com/2016/09/connect-to-a-wireless-network-using-command-line-nmcli/)
        ```
        nmcli device wifi rescan
        nmcli device wifi list
        nmcli device wifi connect <SSID-Name> password <wireless-password>
        ```
3. Enable SSH
    ```
    sudo systemctl enable ssh
    sudo systemctl start ssh
    ```
4. Configure Host
    * We will give our device a static ip so it keeps the ip between reboots.
        * `cat >> /etc/dhcpcd.conf`
        * Find your device IP. It will be in the form of x.x.x.x. You can find your device ip (on a mac) using `ifconfig | grep inet`. My mac ip was 192.168.0.26 so my ip format is like 192.168.0.x.
        * Paste the following code block
            ```
            interface eth0
            static ip_address=x.x.x.y/24
            static routers=x.x.x.1
            static domain_name_servers=8.8.8.8
            
            where, x.x.x.x is same as your device ip and y is the ip you want and reboot the machine
            ```
5. Install Docker
     ```
     curl -sSL get.docker.com | sh && \
     sudo usermod pi -aG docker && \
     newgrp docker
     ```
6. Disable SWAP
    * First set `CONF_SWAPSIZE=0` in `/etc/dphys-swapfile` and then execute the following command
    ```
     sudo dphys-swapfile swapoff && \
     sudo dphys-swapfile uninstall && \
     sudo update-rc.d dphys-swapfile remove
     ```
    * Following command should return empty to verfiy swap is disabled
        `sudo swapon --summary`
7. Edit the `/boot/cmdline.txt` file. Add the following in the end of the file. This needs to be in the same line as all the other text in the file. Do not create a new file.
    * `cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`
    * Reboot the system
8. Install k8s
    * Create the following file: `/etc/apt/sources.list.d/kubernetes.list` and add the following in the file
        `deb http://apt.kubernetes.io/ kubernetes-xenial main`
        
    * Execute the following as root:
        `curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -`

You may encounter: `Warning: apt-key is deprecated. Manage keyring files in trusted.gpg.d instead (see apt-key(8)).`

To resolve the warning, you need to move your GPG public keys from the `/etc/apt/trusted.gpg` file to the `/etc/apt/trusted.gpg.d` directory.
```
To do this, follow these steps:

1. Open a terminal window.
2. Change to the APT directory:

cd /etc/apt

3. Copy the trusted.gpg file to the trusted.gpg.d directory:

sudo cp trusted.gpg trusted.gpg.d

4. Remove the trusted.gpg file:

sudo rm trusted.gpg

5. Update the APT cache:

sudo apt update

```
Once you have completed these steps, the warning should disappear when you run `apt-key` commands.

**Note:** If you have any GPG public keys in the `/etc/apt/keyrings/` directory, you can leave them there. The APT package manager will automatically search that directory for keys when verifying packages.

**Why is apt-key deprecated?**
The apt-key command is deprecated because it is not as secure as the newer gpg command. The apt-key command does not support all of the latest security features, and it can be used to accidentally trust keys for all repositories on your system.

The gpg command is a more secure and flexible way to manage GPG public keys. The APT package manager now uses the gpg command to verify packages, and it is recommended that you use the gpg command to manage your GPG public keys as well.

Update the packages

`sudo apt-get update`


Install kubelet, kubeadm, and kubectl:

`sudo apt-get install -y kubelet kubeadm kubectl`


**On the master node:**

Initialize the Kubernetes cluster (The following may take about 15 mins to complete.
):

`sudo kubeadm init --token-ttl=0 --pod-network-cidr=10.244.0.0/16`

Note: If you get the error `[ERROR CRI]: container runtime is not running`, see the troubleshooting guide below.


Above command will produce output similar to the following to be run on worker nodes to join the cluster(this is not required on a single node cluster:
```
kubeadm join <master-ip>:6443 --token <token> \
--discovery-token-ca-cert-hash sha256:<hash>
```
Incase you lose this line or can’t find in the terminal, you can find the token in the following way. `kubeadm token list` will show you all available token. From there you can find the token. 

Finding the sha hash is a bit trickier. But you can get it by `openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'` This long and convoluted command gets the sha256 for the certificate.

Finally run the following as normal user, not as root:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**On the master node:**
Install network driver (Flannel). Weave may not work on raspberrypi (https://kubernetes.io/docs/concepts/cluster-administration/addons/)
```
sudo sysctl net.bridge.bridge-nf-call-iptables=1
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
To run pods on the master node in a single node cluster setup, remove the taint on the master node:

`kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-`

### Troubleshooting:

1) Getting the following error during `kubeadm init`
   `[ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: /etc/kubernetes/manifests/kube-apiserver.yaml already exists`
   
   **Fix:** First run the kubeadm reset before executing the kubeadm init command

2) Getting following errors during kubeadm reset
  
   a) `port 10251 and 10252 are in use`
    
   **Fix:** check the usage status of all ports. It seems like kubeadm failed to reset the controller and scheduler. (https://github.com/kubernetes/kubeadm/issues/339)
      ```
       $ netstat -lnp | grep 1025
       tcp6       0      0 :::10251                :::*                    LISTEN      4366/kube-scheduler
       tcp6       0      0 :::10252                :::*                    LISTEN      4353/kube-controlle
       $ kill 4366
       $ kill 4353
      ```
   b) `[ERROR CRI]: container runtime is not running` 
      
      **Fix:** (Source: https://www.reddit.com/r/kubernetes/comments/utiymt/kubeadm_init_running_into_issue_error_cri/)
     
      https://github.com/containerd/containerd/issues/4581
              
       $ sudo rm /etc/containerd/config.toml
       $ sudo systemctl restart containerd
       $ sudo kubeadm init
      

Once you have completed these steps, your Kubernetes cluster on Raspberry Pi will be up and running!
