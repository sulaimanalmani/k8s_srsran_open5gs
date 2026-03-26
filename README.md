# k8s_srsran_open5gs
Containerized/kubernetes deployment of E2E 5G testbed using srsRAN and Open5gs

# srsRAN (zmq) + Open5GS Kubernetes Deployment

This guide provides step-by-step instructions for deploying a testbed for srsRAN (zmq) and Open5GS using Kubernetes. This guide builds on [open5gs-k8s](https://github.com/niloysh/open5gs-k8s) and deploy the RAN component (srsRAN) for an E2E 5G testbed. The RAN deployment is a containerized version of the [srsRAN ZMQ deployment](https://docs.srsran.com/projects/project/en/latest/tutorials/source/srsUE/source/index.html#zeromq-based-setup)

## Prerequisites

- Ubuntu 20.04 or newer
- Reasonably powerful machine (8-32 cores), 16-32Gb RAM, 128Gb Disk space

## Step 0: Setup Repository and Environment

1. Clone the repository and add the `bin` directory to your `PATH`:

    ```bash
    git clone https://github.com/sulaimanalmani/k8s_srsran_open5gs.git
    echo 'export PATH="/path/to/k8s_srsran_open5gs/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    cd /path/to/k8s_srsran_open5gs/
    git clone https://github.com/sulaimanalmani/testbed-automator.git
    ```
    
## Step 1: Install Kubernetes and Required Components

Navigate to the `testbed_automator` directory and run the installation script:

```bash
cd /path/to/k8s_srsran_open5gs/testbed_automator/
./install_k8s.sh
```

Make sure kubernetes is up and running using the follwoing commands:
```bash
kubectl get nodes
kubectl get pods -A
```

## Step 2: Deploy Namespaces, Prometheus, Network-Attachment-Definitions, and MongoDB

1. Create required namespaces:

```bash
kubectl create namespace open5gs
kubectl create namespace monitoring
```

2. Install Prometheus for monitoring:
```bash
cd path/to/k8s_srsran_open5gs/config/prometheus/
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -f debug_kube-prometheus-values-nuc.yaml -n monitoring
```

Make sure that monitoring containers are up and running using the follwoing command 
```bash
kubectl get pods -n monitoring
```

For monitoring the kubernetes cluster, you can access graphana at port `32000` using the user:password `admin:prom-operator`. If you are accessing the testbed using ssh, you need to port-forward port 32000 to the server using the following command:

```bash
ssh -L 32000:localhost:32000 user@your-server-ip
```

3. Deploy network attachment definitions:

```bash
cd path/to/k8s_srsran_open5gs/config/open5gs/
kubectl apply -k ./networks5g/ -n open5gs
```

4. Deploy MongoDB:
```bash
cd path/to/k8s_srsran_open5gs/config/open5gs/
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml
kubectl apply -k ./mongodb/ -n open5gs
```

5. Add subscribers using mongo-tools:
```bash
cd path/to/k8s_srsran_open5gs/config/open5gs/mongo-tools/
sudo apt-get install python3.12-venv
python3 -m venv ../venv
source ../venv/bin/activate
pip install bson pymongo
python3 ./modify_subscribers.py add
```

## Step 3: Deploy Open5GS and srsRAN (zmq)

1. Deploy Open5GS:

    ```bash
    cd path/to/k8s_srsran_open5gs/config/open5gs/open5gs/
    kubectl apply -k . -n open5gs
    k8s-log.sh amf open5gs # To view AMF logs
    ```
If the core has been succesfully deployed, you should see logs similar to the following:

![amf_logs](/images/amf_logs.png "AMF Logs.")


2. Deploy srsRAN gNB:

    ```bash
    cd path/to/k8s_srsran_open5gs/config/srsRAN/srsran-gnb/
    kubectl apply -k . -n open5gs
    k8s-shell.sh gnb open5gs
    /srsran/config/start_gnb.sh
    ```

If the gNB successfully connects to the amf, you should see logs similar to the following:

![gnb_logs](/images/gnb_logs.png "gNB Logs.")

    
## Step 4: Deploy UEs and Connect to gNB

1. Deploy UEs:

    ```bash
    cd path/to/k8s_srsran_open5gs/config/ues/srsue/
    kubectl apply -k . -n open5gs
    ```

2. Open multiple terminals for each UE (e.g., `ue1` to `ue10`) and start them:

    ```bash
    k8s-shell.sh ue1 open5gs
    /srsran/config/start_ue.sh <ue_number>
    ```

3. Start GNU Radio for connecting gNB to UEs:

    ```bash
    k8s-shell.sh ue1 open5gs
    /srsran/config/start_gnu.sh <total_ue_number>
    ```

4. After UEs are connected, add the default route:

    ```bash
    k8s-shell ue1 open5gs
    /srsran/config/add_route.sh
    ```
    
If UEs are connected successfully, you should see UE and GNU logs similar to the following:

![gnb_logs](/images/ue_logs.png "UE Logs.")
![gnb_logs](/images/gnu_logs.png "GNU Logs.")

    
## Step 5: Send Traffic Through the Testbed
After the UEs are connected, you should the following logs:


The UE interfaces are created within the respective namespaces (`ue1` to `ue10`), and can be accessed using the 'ip netns' command within the UE container

1. List namespaces for UEs:

    ```bash
    k8s-shell ue1 open5gs
    ip netns list
    ip netns exec ue1 ifconfig
    ```

2. Send traffic through a specific UE (replace `ue1` with the appropriate UE namespace):

    ```bash
    k8s-shell ue1 open5gs
    ip netns exec ue1 ping google.ca
    ```

You can use this method to send traffic from any UE to test connectivity.


## (Optional) Reaching the UEs

To reach the UEs, an `iperf` container is included. You can use this container to create a iperf client / servers and to generate traffic for the UEs. You can deploy the container using the following commands (assuming `10.41.0.2` as the UE IP address)

```bash
cd path/to/k8s_srsran_open5gs/config/iperf/
kubectl apply -k . -n open5gs
k8s-shell.sh iperf open5gs
ping 10.41.0.2
```


## (Optional) Step 6: CPU Resource Scaling

To scale the CPU resources allocated to the gNB process, use the following script (requires Ubuntu 22.04+):

```bash
cd srsran_open5gs/scripts
sh ./set_cgroup_cpu.sh 1500 # Allocates 1500 milli-cores (1.5 CPU)
```

## (Optional) Step 6: Setting up the Transport Network and Bandwidth Scaling

Please refer to the [onos](configs/onos/README.md) section for setting up the transport network and bandwidth scaling.

```bash
cd srsran_open5gs/scripts
sh ./set_bw.sh 10 # Allocates 10 Mbps bandwidth
```

Note that you must have ssh access to the transit node and a password less sudo access for the user on the transit node. For the latter, please refer to [stackoverflow](https://askubuntu.com/questions/147241/execute-sudo-without-password)

## References

The guide is based on the work presented in the following paper:

Sulaiman M., Ahmadi M., Salahuddin M. A., Boutaba R., & Saleh A. (2023). [Generalizable Resource Scaling of 5G Slices using Constrained Reinforcement Learning](https://arxiv.org/abs/2306.09290). arXiv preprint arXiv:2306.09290.
