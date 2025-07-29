# ELK Kubernetes Logging Setup

This repository contains configuration files for setting up an **EFK** (Elasticsearch, Fluentd, Kibana) logging stack on a Kubernetes cluster. The setup enables centralized logging for containerized applications, with Fluentd collecting logs, Elasticsearch storing and indexing them, and Kibana providing a visualization interface.

---

## Prerequisites

- A running Kubernetes cluster (e.g., EKS, GKE, or Minikube).
- `kubectl` configured to interact with your cluster.
- Helm (optional, if you prefer deploying with Helm charts for ECK).
- A node group with the label `nodegroup-type: efk-group` and toleration key `efk-key=true` for Elasticsearch and Kibana deployments.
- Access to a storage class (e.g., `gp2-csi` for AWS EKS) for persistent storage.

---

## Repository Contents

- `fluentd-config.yaml`: ConfigMap defining Fluentd's configuration for log collection and forwarding to Elasticsearch.
- `fluentd.yaml`: Defines a ServiceAccount, ClusterRole, ClusterRoleBinding, and DaemonSet for running Fluentd on each node to collect container logs.
- `kibana-svc.yml`: Service configuration for exposing Kibana via a LoadBalancer on port 5601.
- `02_es.yml`: Elasticsearch configuration for a single-node cluster using version 7.17.24 with a 10Gi persistent volume.
- `03_kibana.yml`: Kibana configuration for version 7.17.24, linked to the Elasticsearch cluster.

---

## Setup Instructions

### Step 1: Prepare the Cluster

1. Ensure your Kubernetes cluster has a node group labeled with:
    ```yaml
    nodeSelector:
      nodegroup-type: efk-group
    tolerations:
      - key: "efk-key"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
    ```
    This should match the `nodeSelector` and `tolerations` in `02_es.yml` and `03_kibana.yml`.

2. Verify that a storage class (e.g., `gp2-csi`) is available for Elasticsearch's persistent volume:
    ```sh
    kubectl get storageclass
    ```

---

### Step 2: Deploy Elasticsearch

1. Apply the Elasticsearch configuration:
    ```sh
    kubectl apply -f 02_es.yml
    ```

2. Verify the Elasticsearch pod is running:
    ```sh
    kubectl get pods -l elasticsearch.k8s.elastic.co/cluster-name=quickstart
    ```

3. Retrieve the Elasticsearch password:
    ```sh
    kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode
    ```

4. Replace `xxxxxxxxxxxxxxxxxxxxxx` in both `fluentd-config.yaml` and `fluentd.yaml` with the password retrieved above.

---

### Step 3: Deploy Kibana

1. Apply the Kibana configuration:
    ```sh
    kubectl apply -f 03_kibana.yml
    ```

2. Apply the Kibana LoadBalancer service:
    ```sh
    kubectl apply -f kibana-svc.yml
    ```

3. Verify the Kibana pod is running:
    ```sh
    kubectl get pods -l kibana.k8s.elastic.co/name=quickstart
    ```

4. Get the external IP for Kibana:
    ```sh
    kubectl get svc kibana-lb -n efklog
    ```

5. Access Kibana via browser:
    ```
    http://<external-ip>:5601
    ```

---

### Step 4: Deploy Fluentd

1. Apply the Fluentd ConfigMap:
    ```sh
    kubectl apply -f fluentd-config.yaml
    ```

2. Apply the Fluentd DaemonSet and associated resources:
    ```sh
    kubectl apply -f fluentd.yaml
    ```

3. Verify Fluentd pods are running on each node:
    ```sh
    kubectl get pods -l app=fluentd
    ```

---

### Step 5: Validate the Logging Pipeline

1. Access Kibana via the LoadBalancer IP.
2. Log in with the `elastic` user and the password retrieved in Step 2.
3. In Kibana, go to **"Stack Management" > "Index Patterns"** and create a new index pattern using:
    ```
    logstash-*
    ```
4. Check the **Discover** tab to view logs being collected from containers.
5. Verify that logs from `/var/log/containers` and `/var/log/pods` are visible.

---

## License

This project is licensed under the MIT License.
