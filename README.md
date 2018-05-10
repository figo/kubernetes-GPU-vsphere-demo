**bring up kubernetes GPU cluster with vSphere**

   
   
This page is to help you bring up a kubernetes cluster with GPU capability on top of vsphere,
so that you could easily deploy machine learning workloads on the cluster,    
two conditions need been satisfied before we move forward:

 1. Physical server with supported NVIDIA GPU installed (AMD can be supported also, but let target NVIDIA on this page)
 2. vSphere (6.5 / 6.7) image ready to install

![arch](https://github.com/figo/kubernetes-GPU-vsphere-demo/blob/master/architecture.png)

   
    
**Here are the detail steps:**
1. Install ESX on host, and install vCenter  
   
   
   
2. Enable GPU directpath I/O on host through vCenter/host client:   
   `https://kb.vmware.com/s/article/1010789`
   
   
    
3. Create VM (ubuntu 16.04) and attach GPU device to it (you will have to reserve all VM memory)            
    `https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.vsphere.vm_admin.doc/GUID-C597DC2A-FE28-4243-8F40-9F8061C7A663.html`
   
   
   
4.  Inside VM: Install GPU driver, install docker, and install nvidia container runtime  
    Get the latest version of driver from nvidia and install it:  
    `http://www.nvidia.com/Download/index.aspx`  
     verify driver version with command: `nvidia-smi`

     install docker  
     Verify docker version with command: `docker version`

     install nvidia container runtime with command:  
     `apt-get install -y nvidia-docker2=2.0.2+docker1.13.1-1 nvidia-container-runtime=1.1.1+docker1.13.1-1`

     Configure docker default runtime to nvidia:  
     vi /etc/docker/daemon.json, make sure it looks like this: 

 
          {
              "default-runtime": "nvidia",  
              "runtimes": {   
                  "nvidia": {   
                      "path": "/usr/bin/nvidia-container-runtime",  
                      "runtimeArgs": []   
                            }  
                          } 
          }
        
      Restart docker:  
      `systemctl daemon-reload`  
      `systemctl restart docker`  
     
      Verify tensorflow container can run successfully before kubernetes involved:  
      (use the appropriate image from https://hub.docker.com/r/tensorflow/tensorflow/tags/)  
      `docker run --runtime=nvidia  -it gcr.io/tensorflow/tensorflow:1.2.0-gpu-py3 python -c 'import tensorflow'`
  
  
  
5.  Create kubernetes cluster:  
    I am using kubeadm and flannel to setup a cluster with one master and 2 worker nodes.  
    `https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/`  
    you could bring up the cluster by other method, it does not matter much, make sure   
    all nodes are ready:  
      `kubectl get nodes`  


    Once we get cluster running, it is time to deploy nvidia device plugin:  
    `kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.10/nvidia-device-plugin.yml`  
    `kubectl get pods -n kube-system`  

     with plugin installed, we should verify GPU been exposed to cluster:  
     `kubectl describe node $node_name | egrep 'Capacity|Allocatable|gpu'`


  
              Capacity:
                     nvidia.com/gpu:     1
              Allocatable:
                     nvidia.com/gpu:     1  

      With this, the k8s cluster is ready to run machine learning workloads.  
  
  
  
6. Build and package the workload through docker commands.  
   Then create a yaml file for the deployment:  


       resources:
         limits:
         nvidia.com/gpu: 2 # requesting 2 GPU
  
  
  
7. Just deploy the pod or service and check the logs

