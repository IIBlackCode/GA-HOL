# GithubAction-SpringBoot-Sample

- Java ver : 17
- Type : Gradle
- Gradle ver : 8.1.1
- SpringBoot ver : 3.1.2
- Template Engine : Thymeleaf
- Packaging : Jar

---

# AKS CICD workflow
## main.yml
```
name: Workflow Call

on:
  push:
    branches: ["master"]
  workflow_dispatch:

jobs:
  buildImage:
    permissions:
      contents: read
      id-token: write
    uses: ./.github/workflows/build.yml
    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
      
  deployToAKS:
    permissions:
      actions: read
      contents: read
      id-token: write
    uses: ./.github/workflows/deploy.yml
    needs: [buildImage]
    secrets:
      AZURE_CREDENTIALS: ${{ secrets.AZURE_CREDENTIALS }}
```

## build.yml
```
name: Build and Push to ACR

on:
  workflow_call:
      secrets:
        AZURE_CREDENTIALS:
          required: true
permissions:
  contents: read
  id-token: write

env:
  AZURE_CONTAINER_REGISTRY: ${{ vars.REGISTRY_NAME }}
  CONTAINER_NAME: ${{ vars.CONTAINER_NAME }}
  RESOURCE_GROUP: ${{ vars.RESOURCE_GROUP_NAME }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # Checks out the repository this file is in
      - uses: actions/checkout@v3

      # Logs in with your Azure credentials
      - name: Azure login
        uses: azure/login@v1
        with:
          creds: '${{ secrets.AZURE_CREDENTIALS }}'

      # Builds and pushes an image up to your Azure Container Registry
      - name: Build and push image to ACR
        run: |
          az acr build --image ${{ env.AZURE_CONTAINER_REGISTRY }}.azurecr.io/${{ env.CONTAINER_NAME }}:${{ github.run_number }} --registry ${{ env.AZURE_CONTAINER_REGISTRY }} -g ${{ env.RESOURCE_GROUP }} ./demo-jar

```

## deploy.yml
```
name: Deploy to AKS

on:
  workflow_call:
      secrets:
        AZURE_CREDENTIALS:
          required: true
permissions:
  actions: read
  contents: read
  id-token: write

env:
  AZURE_CONTAINER_REGISTRY: ${{ vars.REGISTRY_NAME }}
  CONTAINER_NAME: ${{ vars.CONTAINER_NAME }}
  RESOURCE_GROUP: ${{ vars.RESOURCE_GROUP_NAME }}
  CLUSTER_NAME: ${{ vars.CLUSTER_NAME }}
  DEPLOYMENT_MANIFEST_PATH: ${{ vars. DEPLOYMENT_MANIFEST_PATH }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # Checks out the repository this file is in
      - uses: actions/checkout@v3

      # Logs in with your Azure credentials
      - name: Azure login
        uses: azure/login@v1
        with:
          creds: '${{ secrets.AZURE_CREDENTIALS }}'

      # Use kubelogin to configure your kubeconfig for Azure auth
      - name: Set up kubelogin for non-interactive login
        uses: azure/use-kubelogin@v1
        with:
          kubelogin-version: 'v0.0.25'

      # Retrieves your Azure Kubernetes Service cluster's kubeconfig file
      - name: Get K8s context
        uses: azure/aks-set-context@v3
        with:
          resource-group: ${{ env.RESOURCE_GROUP }}
          cluster-name: ${{ env.CLUSTER_NAME }}
          admin: 'false'
          use-kubelogin: 'true'
      
      # Update YAML Image
      - name: Update YAML Image
        run: |
          sed -i 's|acrname.azurecr.io/imagename:v1|${{ env.AZURE_CONTAINER_REGISTRY }}.azurecr.io/${{ env.CONTAINER_NAME }}:${{ github.run_number }}|g' ${{ env.DEPLOYMENT_MANIFEST_PATH }}

      # Deploys application based on given manifest file
      - name: Deploys application
        uses: Azure/k8s-deploy@v4
        with:
          action: deploy
          manifests: ${{ env.DEPLOYMENT_MANIFEST_PATH }}
```
