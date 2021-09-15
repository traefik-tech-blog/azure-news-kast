# Azure News Kast

This repo contains the demo given during the [Azure News Kast episode](https://www.youtube.com/watch?v=pKWcVSZazys&list=PLnQ8_FdCQrSkXgaI4gBownT-RL3jEZCxV)

# Create AKS cluster

```bash
export AZURE_APP_ID=aaaa
export AZURE_TENANT_ID=bbbb
export AZURE_PASSWORD=ccccc
export AZURE_SUBSCRIPTION=your-subscription-id

export AZURE_LOCATION=westeurope
export CLUSTER_NAME=ank
export PREFIX=${CLUSTER_NAME}-$(openssl rand -hex 12 | tr '[:upper:]' '[:lower:]')
export DOMAIN=${PREFIX}.${AZURE_LOCATION}.cloudapp.azure.com
export LE_EMAIL=michael@traefik.io
export KUBECONFIG=~/.kube/${CLUSTER_NAME}.yaml

# Login to azure
az login --service-principal --username ${AZURE_APP_ID} --password ${AZURE_PASSWORD} --tenant ${AZURE_TENANT_ID}

# Create group in azure for the hands on
az group create --name ${CLUSTER_NAME} --location ${AZURE_LOCATION}

# Create aks cluster
az aks create --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME} --node-count 3 --ssh-key-value=~/.ssh/id_rsa.pub --subscription ${AZURE_SUBSCRIPTION} --service-principal ${AZURE_APP_ID} --client-secret ${AZURE_PASSWORD}

# Retrieve aks credentials
az aks get-credentials --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME} --file ~/.kube/${CLUSTER_NAME}.yaml

# Enable metrics
az aks enable-addons -a monitoring --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME}
```

# Traefik

Prerequisites:
- AKS cluster up and ready

## Installation

```bash

# Adapt static configuration in the deployment
sed -i "s/LE_EMAIL/${LE_EMAIL}/g" traefik/03-deployment.yaml
sed -i "s/DOMAIN/${DOMAIN}/g" traefik/03-deployment.yaml

# Deploy Traefik
kubectl apply -f traefik/

# Add annotation to the LoadBalancer service to create DNS entry
kubectl annotate service traefik -n traefik service.beta.kubernetes.io/azure-dns-label-name=${PREFIX}

# Check DNS entry creation
dig ${DOMAIN} # Shoudl return an A entry on the LB IP
```

## Demo

### Exposing the dashboard

Deploy the dashboard.

```bash
# Adapte dashboard ingress
sed -i "s/DOMAIN/${DOMAIN}/g" traefik/dashboard/05-ingress.yaml

k apply -f traefik/dashboard/
```

### Demo App

Deploy the demo application and expose this application with an IngressRoute.

```bash
sed -i "s/DOMAIN/${DOMAIN}/g" app/03-ingress.yaml

kubectl apply -f app/
```

With the application deployed, we can now add the following items:
- Middlewares:
  - Custom header
  - Rate limiting

## Clean

Don't forget to stop your environment.

```bash
# Delete AKS cluster
az aks delete --name ${CLUSTER_NAME} --resource-group ${CLUSTER_NAME} --yes

# Delete group
az group delete --name ${CLUSTER_NAME} --yes

# Delete the ad app
az ad app delete --id ${AD_APP_ID}
```
