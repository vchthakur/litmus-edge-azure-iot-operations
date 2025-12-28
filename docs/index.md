Overview:

This guide documents how to integrate Litmus Edge with Azure IoT Operations by securely streaming data from Litmus Edge into the Azure IoT Operations MQTT broker for further processing, modeling, and analytics.

The focus of this guide is a production-oriented integration that aligns with Azure Arc–enabled Kubernetes deployments and enterprise OT security practices.

Architecture:

# Integrating Litmus Edge with Azure IoT Operations

## Overview

This guide documents a practical integration of **Litmus Edge** with **Azure IoT Operations**, focusing on securely streaming industrial data from Litmus Edge into the Azure IoT Operations MQTT broker for further processing, modeling, and analytics.

The intent of this document is to provide **production-oriented guidance** rather than a simple proof-of-concept. It highlights architectural decisions, security considerations, and operational constraints that are important when running this integration in real industrial environments.

---

## Architecture

![Architecture](../media/Azure_IoT_Operations_with_LitmusEdge_overview_architecture.png)

Azure IoT Operations runs on **Azure Arc–enabled Kubernetes clusters**, providing a consistent control plane for edge workloads and centralized governance.

Litmus Edge supports container-based deployment but is not fully Kubernetes-native. Despite this, deploying Litmus Edge **as a pod within the same Kubernetes cluster as Azure IoT Operations** is a practical and clean approach:

- Reduced network complexity  
- Clear separation of responsibilities  
- Alignment with platform lifecycle and governance  

In this setup:
- **Litmus Edge** handles OPC UA connectivity, normalization, and contextualization close to the machines  
- **Azure IoT Operations** manages MQTT, schema registration, asset modeling, and downstream data flows  

MQTT acts as the explicit integration boundary between OT systems and the platform.

---

## Prerequisites

Before starting, ensure the following prerequisites are met:

- A working **Azure IoT Operations** deployment  
- Access to the **Litmus Portal** to obtain:
  - Litmus Edge container image  
  - Service account credentials (`litmus-se-read-json`)  
- Access to a private container registry (recommended for production use)  
- `kubectl` and `helm` installed and configured  

---

## Integration Flow

The integration is performed in two main stages:

1. Deploy Litmus Edge into the Azure Arc–enabled Kubernetes cluster  
2. Establish a secure MQTT connection to Azure IoT Operations  

---

## Step 1 – Deploy Litmus Edge to Kubernetes

Litmus Edge does not currently provide a fully Kubernetes-native deployment model. While basic Kubernetes installation documentation exists, additional considerations are required for production deployments.

### Container Image Handling

The Litmus Edge container image is hosted in a **private registry** and is tagged as `latest`. Using the `latest` tag directly is not recommended, as container restarts may introduce unintended upgrades.

A more controlled approach is to:
- Import the image into a private container registry (for example, Azure Container Registry)
- Apply an explicit version tag

Example (shown for Litmus Edge version `3.16.2`):

```bash
$litmusRepoPassword = Get-Content ./litmus-se-read-json

az acr import \
  --name <acr-name> \
  --source us-docker.pkg.dev/litmus-sales-enablement/litmusedge/litmusedge-std-docker:latest \
  --image litmusedge/litmusedge-std-docker:3.16.2 \
  --username _json_key \
  --password $litmusRepoPassword

```
Kubernetes Namespace and Registry Credentials
Create a dedicated namespace and registry secret:

```bash
kubectl create namespace le-prod

$litmusRepoPassword = Get-Content ./litmus-se-read-json
kubectl create secret docker-registry litmus-credential \
  --docker-server=us-docker.pkg.dev \
  --docker-username=_json_key \
  --docker-password=$litmusRepoPassword \
  -n le-prod

```
If a private registry is used, adjust the secret configuration accordingly.

Deploying via Helm

Litmus provides YAML-based deployment examples but does not currently ship an official Helm chart. For improved lifecycle management, a community Helm-based deployment can be used.
Add the Helm repository:

```bash
helm repo add jmayrbaeurl https://raw.githubusercontent.com/jmayrbaeurl/helmchart-repo/master/index
helm repo update
```
Deploy Litmus Edge:
```bash
helm upgrade --install litmusedge jmayrbaeurl/litmusedge \
  -n le-prod \
  --create-namespace
```
To expose the Litmus Edge UI externally (optional):
```
helm upgrade --install litmusedge jmayrbaeurl/litmusedge \
  -n le-prod \
  --create-namespace \
  --set service.externalAccess=true
```
Note:
The Helm deployment creates a PersistentVolumeClaim for data storage.
This volume is not deleted automatically when uninstalling the Helm release.

Step 2 – Connect Litmus Edge to Azure IoT Operations
Once Litmus Edge is running, the next step is to stream data securely into Azure IoT Operations.
Litmus Edge provides an MQTT – Generic over SSL connector that can be used to connect to the Azure IoT Operations MQTT broker.

MQTT Listener and Authentication Strategy
Azure IoT Operations exposes multiple MQTT listeners. This integration uses the default internal listener, which:
Supports TLS
Is required for internal Azure IoT Operations components
Rather than modifying the default listener, an additional X.509 authentication policy is added. This is necessary because Litmus Edge does not currently support Kubernetes ServiceAccount authentication for MQTT connections.

Certificate Preparation

Extract the Azure IoT Operations MQTT broker CA certificate:
```
kubectl get configmap azure-iot-operations-aio-ca-trust-bundle \
  -n azure-iot-operations \
  -o "jsonpath={.data['ca\.crt']}" > ca.cert
```

Generate client-side certificates for Litmus Edge using the step CLI:
```
step certificate create "Litmus Edge Root CA" \
  litmusedge_root_ca.crt litmusedge_root_ca.key \
  --profile root-ca --no-password --insecure

step certificate create "Litmus Edge Intermediate CA 1" \
  litmusedge_intermediate_ca.crt litmusedge_intermediate_ca.key \
  --profile intermediate-ca \
  --ca litmusedge_root_ca.crt \
  --ca-key litmusedge_root_ca.key \
  --no-password --insecure

step certificate create aiomqttconnector \
  aiomqttconnector.crt aiomqttconnector.key \
  --ca litmusedge_intermediate_ca.crt \
  --ca-key litmusedge_intermediate_ca.key \
  --bundle --not-after 2400h \
  --no-password --insecure

```

Register the Litmus Edge CA with Azure IoT Operations:
```
kubectl create configmap litmusedge-ca \
  -n azure-iot-operations \
  --from-file=client_ca.pem=litmusedge_root_ca.crt
```
![Architecture](../media/authpolicies.png)

Authentication Policy Configuration

Add an X.509 authentication policy to the default listener (port 18884) using the following configuration:
```
{
  "authorizationAttributes": {
    "intermediate": {
      "attributes": {
        "manufacturer": "litmus edge"
      },
      "subject": "CN = Litmus Edge Intermediate CA 1"
    },
    "connector": {
      "attributes": {
        "group": "connector_group"
      },
      "subject": "CN = aiomqttconnector"
    }
  },
  "trustedClientCaCert": "litmusedge-ca"
}

```

Configuring the Litmus Edge Connector
![Architecture](../media/defaultlistener.png)

In the Litmus Edge UI:

1. Navigate to Integration → Streaming
2. Create a new connector of type MQTT over SSL
3. Leave Client ID, username, and password empty
4. Upload the client certificate and key
5. Save the configuration

A successful connection is indicated by a green status.
![Architecture](../media/leconnector.png)
![Architecture](../media/connectorgreen.png)
![Architecture](../media/managetopics.png)
![Architecture](../media/mqttexplorer.png)
Verifying Data Flow
Configure topic publishing in Litmus Edge and verify MQTT traffic using a tool such as MQTT Explorer. Topics should be visible via the Azure IoT Operations MQTT broker.

Operational Considerations

1. Litmus Edge is monolithic and stateful
2. Horizontal scaling is not supported
3. Vertical scaling must be planned explicitly
4. Certificate lifecycle management requires operational ownership

These constraints should be understood and accepted before production rollout.

Conclusion
Litmus Edge provides strong capabilities for industrial data transformation and contextualization at the edge. Running it alongside Azure IoT Operations on Kubernetes is a practical and effective integration pattern when its constraints are clearly understood.
When deployed intentionally, this architecture represents one of the more robust OT-to-cloud integration approaches available today for brownfield industrial environments.
