### Quickstart for Google Container Engine

logon to GCE console from [https://console.cloud.google.com](https://console.cloud.google.com). 

Use Google Cloud Shell to interact with `kubectl` immediately, use the Google Cloud Shell, which comes preinstalled with the `gcloud` and `kubectl` command-line tools. Follow these steps to launch the Google Cloud Shell:

![Google Cloud Shell](/img/shell_icon.png?raw=true)

Go to [Google Cloud Platform Console](https://console.cloud.google.com/?_ga=1.126719242.621469569.1491942438). Click the **Activate Google Cloud Shell** button at the top of the console window. A Cloud Shell session opens inside a new frame at the bottom of the console and displays a command-line prompt.

![Google Cloud Platform console](/img/new-console.png?raw=true)


Set a default [Compute Engine zone](https://cloud.google.com/compute/docs/zones#available). The following command sets the default zone as `us-central1-b`: 

    gcloud config set compute/zone us-central1-b

You can view your defaults in the `gcloud` command-line tool by running the following command: 

    gcloud config list

 Provision a Kubernetes ClusterFollow these steps to create a Kubernetes cluster in GCE. First Set a default [Compute Engine zone](https://cloud.google.com/compute/docs/zones#available). The following command sets the default zone as `us-central1-b` in Google Cloud Shell. 

    gcloud config set compute/zone us-central1-b

 Create a new project named `cpx-project` by y clicking "CREATE PROJECT" under "All Projects"

![](/img/create-project.png?raw=true)
 
 Under **Home**   
![](/img/home.png?raw=true) 

click on **Go to APIs overview**

![](/img/APIs.png?raw=true)  

and click on **Enable API.**

![](/img/enable-api.png?raw=true)

and then navigate to **Container Enginer API.**

![](/img/container-engine-api.png?raw=true)  

And then click **Enable**

![](/img/enable-api-1.png?raw=true)

You may have to wait a few moments until the Google Cloud Shell recognizes the enabled API setting. To create a cluster enter the following in Google Cloud Shell. This step can take a few minutes to complete

    gcloud container clusters create cpx-demo

Under Google Compute Engine, you can see your virtual machines that make up your kube cluster be provisioned. 

![](/img/vm-instances.png?raw=true) 

Ensure kubectl has authentication credentials. Agree and follow the instructions to retrieve the validation code. 

    gcloud auth application-default login

Enter the following to validate kubectl is working properly:

    kubectl version   
    kubectl cluster-info  
    kubectl</span> get nodes 

Note down your Kubernetes Master URL from the `kubectl cluster-info` command. Mine showed that the Kubernetes master was running at https://104.198.206.231

### Provision CPX for KubeProxy in Kubernetes Cluster

> Note that the following steps from here on are not unique to Kubernetes in GKE. The following instructions are applicable to any Kuberentes deployment where CPX is desired to function with the kube proxy role for internal microservices load-balancing. 

#### Step One: Attach label to the node

Run `kubectl get nodes` to get the names of your cluster’s nodes. Pick out the one that you want to add a label to. In our case, we will assign a lable to all nodes to specify all nodes in the cluster will host CPX.Run `kubectl label nodes <node-name> <label-key>=<label-value>` to add a label to the nodes you’ve chosen. For example,  I ran 

    $ kubectl get nodes
    >>
    NAME                                     STATUS     AGE     VERSION
    gke-cpx-demo-default-pool-509e7c0e-bds7  Ready      44m     v1.5.6
    gke-cpx-demo-default-pool-509e7c0e-fh7b  Ready      44m     v1.5.6
    gke-cpx-demo-default-pool-509e7c0e-lr0q  Ready      44m     v1.5.6

#### Step Two: Create a CPX [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

Create the yaml file in the editor of your choice which will be used to deploy cpx pod on each node with the lable `nscpx`.Save the file `nscpx.yaml` to `/tmp` directory. Here is the [nscpx.yaml](https://bookworm.americasreadiness.com/attachments/2) which holds the contents of our deamonset.

> Remeber to update the **kubernetes_url** value in the nscpx.yaml file to your Kubernetes API URL noted from above. 

    $ cd /tmp  
    $ nano nscpx.yaml   
    # copy and paste file contents

Now create the cpx deamonset using kubectl.

     kubectl create -f nscpx.yaml

 You can validate the damonsets were created successfully by entering the command: 

    $ kubectl get daemonset --namespace=kube-system  
    >>
    **NAME   DESIRED CURRENT READY UP-TO-DATE AVAILABLE NODE-SELECTOR AGE**  
    nscpx       3        3      3       0         0       proxy=nscpx  20m

 You can also see all the containers running across your cluster via the following command: 

    $ kubectl get pods --all-namespaces  
    >>  
    **NAMESPACE    NAME      READY STATUS  RESTARTS AGE**  
    kube-system  fluentd-cloud-logging-gke-cpx-demo-default-pool-509e7c0e-bds7  1/1  Running     0     1h  
    kube-system fluentd-cloud-logging-gke-cpx-demo-default-pool-509e7c0e-fh7b   1/1  Running     0     1h  
    kube-system fluentd-cloud-logging-gke-cpx-demo-default-pool-509e7c0e-lr0q   1/1  Running     0     1h  
    kube-system heapster-v1.2.0.1-1382115970-wh4fm                              2/2  Running     0     1h  
    kube-system kube-dns-2185667875-dt6h3                                       4/4  Running     0     1h  
    kube-system kube-dns-autoscaler-395097547-2kqk6                             1/1  Running     0     1h  
    kube-system kube-proxy-gke-cpx-demo-default-pool-509e7c0e-bds7              1/1  Running     0     1h  
    kube-system kube-proxy-gke-cpx-demo-default-pool-509e7c0e-fh7b              1/1  Running     0     1h  
    kube-system kube-proxy-gke-cpx-demo-default-pool-509e7c0e-lr0q              1/1  Running     0     1h  
    kube-system kubernetes-dashboard-3543765157-x6fj3                           1/1  Running     0     1h  
    kube-system l7-default-backend-2234341178-246vm                             1/1  Running     0     1h  
    kube-system nscpx-f6ng1                                                     1/1  Running     0     23s  
    kube-system nscpx-g8mhk                                                     1/1  Running     0     23s  
    kube-system nscpx-kcbxl                                                     1/1  Running     0     23s

To validate, you can SSH into any host in your cluster and execute CLI commands on the CPX to show all LB vServers automatically configured for your services running on the Kubernetes cluster. First SSH into a Kubernetes minion via Google Cloud Console under Compute Engine. 

![](/img/ssh-instance.png?raw=true)

You will then be connecting to the VM's console directly where you can enter in the `docker ps` command to show all running containers on the host. Container **4b2ec1ed8cd6** is the CPX container we provisioned on this host via daemonset.

    $ docker ps   
    >>  
    **CONTAINER ID IMAGE                                                                     COMMAND               CREATED          STATUS          PORTS     NAMES**  
    4b2ec1ed8cd6 registry.americasreadiness.com:5000/cpx:11.1-48.10                       "/bin/sh -c -- 'bash " 58 minutes ago   Up 58 minutes            k8s_nscpx.d8e54545_nscpx-g8mhk_kube-system_1943408e-1f0a-11e7-9341-42010a800fc1_ec1529ba  
    f95860685be0 gcr.io/google_containers/pause-amd64:3.0                                  "/pause"               58 minutes ago   Up 58 minutes            k8s_POD.d8dbe16c_nscpx-g8mhk_kube-system_1943408e-1f0a-11e7-9341-42010a800fc1_d9583291  
    8852b25f2c25 gcr.io/google_containers/exechealthz-amd64:1.2                            "/exechealthz '--cmd=" 2 hours ago      Up 2 hours               k8s_healthz.eb183f86_kube-dns-2185667875-dt6h3_kube-system_b0bfe64c-1efa-11e7-9341-42010a800fc1_f063ec6f  
    7f820710814c gcr.io/google_containers/dnsmasq-metrics-amd64:1.0.1                      "/dnsmasq-metrics --v" 2 hours ago      Up 2 hours               k8s_dnsmasq-metrics.53491f17_kube-dns-2185667875-dt6h3_kube-system_b0bfe64c-1efa-11e7-9341-42010a800fc1_1bbd304e  
    9018ef6e8b87 gcr.io/google_containers/kube-dnsmasq-amd64:1.4.1                         "/usr/sbin/dnsmasq --" 2 hours ago      Up 2 hours               k8s_dnsmasq.69d68a92_kube-dns-2185667875-dt6h3_kube-system_b0bfe64c-1efa-11e7-9341-42010a800fc1_98073436  
    a7ff0d1d2a01 gcr.io/google_containers/kubedns-amd64:1.9                                "/kube-dns --domain=c" 2 hours ago      Up 2 hours               k8s_kubedns.6545a240_kube-dns-2185667875-dt6h3_kube-system_b0bfe64c-1efa-11e7-9341-42010a800fc1_30a3f5c2  
    c266e6fe4739 gcr.io/google_containers/kubernetes-dashboard-amd64:v1.5.0                "/dashboard --port=90" 2 hours ago      Up 2 hours               k8s_kubernetes-dashboard.5396cb7a_kubernetes-dashboard-3543765157-x6fj3_kube-system_b09b9a81-1efa-11e7-9341-42010a800fc1_38517eac  
    7a8fad6fe86a gcr.io/google_containers/cluster-proportional-autoscaler-amd64:1.1.0-r2   "/cluster-proportiona" 2 hours ago      Up 2 hours               k8s_autoscaler.7ec7f134_kube-dns-autoscaler-395097547-2kqk6_kube-system_b0dcd703-1efa-11e7-9341-42010a800fc1_3f280e37  
    90068b92cc09 gcr.io/google_containers/defaultbackend:1.0                               "/server"              2 hours ago      Up 2 hours               k8s_default-http-backend.a0d6c824_l7-default-backend-2234341178-246vm_kube-system_b05f5a09-1efa-11e7-9341-42010a800fc1_09892c48  
    6f4152df6a61 gcr.io/google_containers/fluentd-gcp:1.28.2                               "/bin/sh -c 'rm /lib/" 2 hours ago      Up 2 hours               k8s_fluentd-cloud-logging.6aa6c538_fluentd-cloud-logging-gke-cpx-demo-default-pool-509e7c0e-bds7_kube-system_51229922e92873f29e001ebdfb62368e_86467c9f  
    def37dd41b1f gcr.io/google_containers/pause-amd64:3.0                                  "/pause"               2 hours ago      Up 2 hours               k8s_POD.d8dbe16c_kube-dns-autoscaler-395097547-2kqk6_kube-system_b0dcd703-1efa-11e7-9341-42010a800fc1_8afe9e62  
    cf9dfb969a32 gcr.io/google_containers/pause-amd64:3.0                                  "/pause"               2 hours ago      Up 2 hours               k8s_POD.d8dbe16c_kube-dns-2185667875-dt6h3_kube-system_b0bfe64c-1efa-11e7-9341-42010a800fc1_510a06b7  
    1b5fd6ba2787 gcr.io/google_containers/pause-amd64:3.0                                  "/pause"               2 hours ago      Up 2 hours               k8s_POD.d8dbe16c_kubernetes-dashboard-3543765157-x6fj3_kube-system_b09b9a81-1efa-11e7-9341-42010a800fc1_9aeaa5d5  
    04daac6decde gcr.io/google_containers/pause-amd64:3.0                                  "/pause"               2 hours ago      Up 2 hours               k8s_POD.d8dbe16c_l7-default-backend-2234341178-246vm_kube-system_b05f5a09-1efa-11e7-9341-42010a800fc1_c1968ea8  
    25b6193e591a gcr.io/google_containers/pause-amd64:3.0                                  "/pause"               2 hours ago      Up 2 hours               k8s_POD.d8dbe16c_fluentd-cloud-logging-gke-cpx-demo-default-pool-509e7c0e-bds7_kube-system_51229922e92873f29e001ebdfb62368e_2c4674d6  
    57adef406f04 gcr.io/google_containers/kube-proxy:cf03177cc1a2711175fc70c089ff7474      "/bin/sh -c 'kube-pro" 2 hours ago      Up 2 hours               k8s_kube-proxy.8c732f13_kube-proxy-gke-cpx-demo-default-pool-509e7c0e-bds7_kube-system_5ef494bd55d9e4874f214ba27f3cb436_91d02bb4  
    77ee96105cec gcr.io/google_containers/pause-amd64:3.0                                  "/pause"               2 hours ago      Up 2 hours               k8s_POD.d8dbe16c_kube-proxy-gke-cpx-demo-default-pool-509e7c0e-bds7_kube-system_5ef494bd55d9e4874f214ba27f3cb436_773c7fb5</pre>
    
You can enter the container's CLI and check via NS CLI commands configured Load Balancing vServers via the following steps. Here you will see on this node that CPX has picked up the Kubernetes infrastructure containers. Next we will provsion additional services and see additional lb vservers automatically be detected in CPX
    
    docker exec -it 4b2ec1ed8cd6 /bin/bash
    >>   
    # **cli_script.sh "sh lb vservers"**   
    >>  
    exec: sh lb vservers  
    1) default-http-backend.kube-system (192.168.1.2:29999) - TCP Type: ADDRESS   
     State: UP  
     Last state change was at Tue Apr 11 22:58:09 2017  
     Time since last state change: 0 days, 01:09:05.150  
     Effective State: UP  
     Client Idle Timeout: 9000 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Listen Policy: NONE  
     Traffic Domain: 0  
    2) heapster.kube-system (192.168.1.2:29998) - TCP Type: ADDRESS   
     State: UP  
     Last state change was at Tue Apr 11 22:58:10 2017  
     Time since last state change: 0 days, 01:09:04.950  
     Effective State: UP  
     Client Idle Timeout: 9000 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Traffic Domain: 0  
    3) kube-dns.kube-system-dns (192.168.1.2:29997) - UDP Type: ADDRESS   
     State: UP  
     Last state change was at Tue Apr 11 22:58:10 2017  
     Time since last state change: 0 days, 01:09:04.750  
     Effective State: UP  
     Client Idle Timeout: 120 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Traffic Domain: 0  
    4) kube-dns.kube-system-dns-tcp (192.168.1.2:29996) - TCP Type: ADDRESS   
     State: UP  
     Last state change was at Tue Apr 11 22:58:10 2017  
     Time since last state change: 0 days, 01:09:04.530  
     Effective State: UP  
     Client Idle Timeout: 9000 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Traffic Domain: 0  
    5) kubernetes-dashboard.kube-system (192.168.1.2:29995) - TCP Type: ADDRESS   
     State: UP  
     Last state change was at Tue Apr 11 22:58:10 2017  
     Time since last state change: 0 days, 01:09:04.320  
     Effective State: UP  
     Client Idle Timeout: 9000 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Traffic Domain: 0

#### Step Three: Deploy additional services on Kubernetes

Now we will deploy additional pods and services on Kubernetes and witness them be automatically detected and configured on CPX. First we will create our yaml file defining replica-controllers for the desired blog site. Use nano or your desired text editor to create [cpx-blog-rc.yaml](https://bookworm.americasreadiness.com/attachments/5) file in `/tmp`. Then create a replication controller with the following command using kubctl on our Google Cloud Shell.

    $ kubectl create -f /tmp/cpx-blog-rc.yaml   
    >>  
    replicationcontroller "cpx-blog" created  

#### See the pods that were created  
    $ kubectl get rc  
    >>  
    NAME             READY  STATUS  RESTARTS AGE  
    cpx-blog-07v52    1/1   Running     0     9s  
    cpx-blog-6xk62    1/1   Running     0     9s  
    cpx-blog-plz8c    1/1   Running     0     9s

Now we will create our yaml file defining container services hosting a blog site. Use nano or your desired text editor to create [cpx-blog-srvc.yaml](https://bookworm.americasreadiness.com/attachments/4) file in `/tmp`.  Then create a replication controller with the following command using kubctl on our Google Cloud Shell.

    $ kubectl create -f cpx-blog-srvc.yaml   
    >>  
    service "cpx-blog" created  

# See the services running in the cluster  
    $ kubectl get services  
    >>  
    NAME         CLUSTER-IP    EXTERNAL-IP  PORT(S)   AGE  
    cpx-blog    10.115.248.90    <none>       80/TCP  11s  
    kubernetes  10.115.240.1     <none>       443/TCP  1d

Now if you SSH back into any of the minons in the Kubernetes Cluster, you can again enter the CPX CLI as prior to show additional configured lb vservers and witness that the cpx-blog containers were automatically configured. 

>Note that vserver #6 is the additional vserver autodiscovered and configured.

    $ **docker exec -it 4b2ec1ed8cd6 /bin/bash**  
    >>   
    # **cli_script.sh "sh lb vservers"**   
    >>  
    exec: sh lb vservers  
    1) default-http-backend.kube-system (192.168.1.2:29999) - TCP Type: ADDRESS   
     State: UP  
     Last state change was at Thu Apr 13 19:28:25 2017  
     Time since last state change: 0 days, 00:32:58.770  
     Effective State: UP  
     Client Idle Timeout: 9000 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Traffic Domain: 0  
    2) heapster.kube-system (192.168.1.2:29998) - TCP Type: ADDRESS   
     State: UP  
     Last state change was at Thu Apr 13 19:28:25 2017  
     Time since last state change: 0 days, 00:32:58.540  
     Effective State: UP  
     Client Idle Timeout: 9000 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Traffic Domain: 0  
    3) kube-dns.kube-system-dns (192.168.1.2:29997) - UDP Type: ADDRESS   
     State: UP  
     Last state change was at Thu Apr 13 19:28:25 2017  
     Time since last state change: 0 days, 00:32:58.330  
     Effective State: UP  
     Client Idle Timeout: 120 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Traffic Domain: 0  
    4) kube-dns.kube-system-dns-tcp (192.168.1.2:29996) - TCP Type: ADDRESS   
     State: UP  
     Last state change was at Thu Apr 13 19:28:25 2017  
     Time since last state change: 0 days, 00:32:58.110  
     Effective State: UP  
     Client Idle Timeout: 9000 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Listen Policy: NONE  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Traffic Domain: 0  
    5) kubernetes-dashboard.kube-system (192.168.1.2:29995) - TCP Type: ADDRESS   
     State: UP  
     Last state change was at Thu Apr 13 19:28:26 2017  
     Time since last state change: 0 days, 00:32:57.870  
     Effective State: UP  
     Client Idle Timeout: 9000 sec  
     Down state flush: ENABLED  
     Disable Primary Vserver On Down : DISABLED  
     Appflow logging: ENABLED  
     No. of Bound Services : 1 (Total) 1 (Active)  
     Configured Method: ROUNDROBIN BackupMethod: NONE  
     Mode: IP  
     Persistence: NONE  
     Connection Failover: DISABLED  
     L2Conn: OFF  
     Skip Persistency: None  
     Listen Policy: NONE  
     IcmpResponse: PASSIVE  
     RHIstate: PASSIVE  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0  
     Mac mode Retain Vlan: DISABLED  
     DBS_LB: DISABLED  
     Process Local: DISABLED  
     Traffic Domain: 0  
    6) cpx-blog.default (192.168.1.2:29994) - TCP Type: ADDRESS**   
     State: UP**  
     Last state change was at Thu Apr 13 19:57:49 2017**  
     Time since last state change: 0 days, 00:03:34.30**  
     Effective State: UP**  
     Client Idle Timeout: 9000 sec**  
     Down state flush: ENABLED**  
     Disable Primary Vserver On Down : DISABLED**  
     Appflow logging: ENABLED**  
     No. of Bound Services : 3 (Total) 3 (Active)**  
     Configured Method: ROUNDROBIN BackupMethod: NONE**  
     Mode: IP**  
     Persistence: NONE**  
     Connection Failover: DISABLED**  
     L2Conn: OFF**  
     Skip Persistency: None**  
     Listen Policy: NONE**  
     IcmpResponse: PASSIVE**  
     RHIstate: PASSIVE**  
     New Service Startup Request Rate: 0 PER_SECOND, Increment Interval: 0**  
     Mac mode Retain Vlan: DISABLED**  
     DBS_LB: DISABLED**  
     Process Local: DISABLED**  
     Traffic Domain: 0

### Conclusion

Here we showed that CPX can take on the built in kube-proxy role to provide contianer-container load balancing within your kuberenetes environment. Additional integrations with NetScaler MAS will provide flexible managibility and most importantly, the potential to collect analytics on application traffic and network traffic across your microservices.
