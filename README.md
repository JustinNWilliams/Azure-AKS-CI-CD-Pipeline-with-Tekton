#  Deploying a CI/CD Pipeline with Tekton on Azure Kubernetes Service (AKS)
![image](https://github.com/user-attachments/assets/e36506d2-d68d-45aa-8f04-20aeea29dd76)

## **Overview**

In this project, we will set up a **CI/CD pipeline** using **Tekton Pipelines** on an **Azure Kubernetes Service (AKS)** cluster. The pipeline automates the process of cloning a repository, building a Node.js application, and preparing it for deployment.

## **Skills You’ll Learn:**
- Setting up and managing Azure Kubernetes Service.
- Installing and configuring Tekton Pipelines.
- Writing Tekton Task and Pipeline YAML files.
- Building and deploying containerized applications with Tekton.

## **Prerequisites**

   - An Azure account: To create and manage the Kubernetes cluster.
   - Azure CLI, `kubectl`, and Tekton CLI (`tkn`): These tools allow you to interact with Azure, Kubernetes, and Tekton from your terminal.
   - Basic familiarity with YAML files: Tekton Pipelines are configured using YAML.

 **Tools to Install**:
   - **Azure CLI**: [Install Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
   - **kubectl**: [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
   - **tkn CLI**: [Install Tekton CLI](https://github.com/tektoncd/cli/releases)

**GitHub**: A GitHub account to clone repositories and host your project.

---

## **Step 1: Set Up Azure Kubernetes Service (AKS)**

### **What is AKS?**
Azure Kubernetes Service (AKS) is a managed Kubernetes service provided by Microsoft Azure. It simplifies Kubernetes management by automating tasks like upgrades, scaling, and monitoring.

### **Steps to Set Up AKS**

1. **Log in to Azure Portal**  
   Go to the [Azure Portal](https://portal.azure.com) and log in.

2. **Create a Resource Group**  
   A **resource group** is a container in Azure that holds resources like your AKS cluster.  
   - Navigate to **Resource Groups** → **Create**.
   - Name it `tekton-rg` and select a region (e.g., `East US`).  
   - **Why?** Resource groups help organize and manage resources efficiently.

![Screenshot 2025-02-28 014942](https://github.com/user-attachments/assets/650c4012-946d-4781-87dd-996bd5a32e52)

3. **Create an AKS Cluster**  
   - Navigate to **Azure Kubernetes Service** → **Create**.
   - Name the cluster `tekton-aks`.
   - Set up **2 nodes** using `Standard_DS2_v2` (adjust as needed).  
   - **Why?** AKS provides a Kubernetes environment for deploying our pipeline.

![Screenshot 2025-02-28 015713](https://github.com/user-attachments/assets/6d7d1762-27c4-406d-a19b-db3dd654b29d)


4. **Connect to the AKS Cluster**  
   Use Azure CLI to connect to your AKS cluster:
   ```bash
   az login
   az aks get-credentials --resource-group tekton-rg --name tekton-aks
   kubectl get nodes
   ```
   - The `get-credentials` command configures `kubectl` to interact with your AKS cluster.
  
![Screenshot 2025-02-28 015805](https://github.com/user-attachments/assets/9f70b192-dfe8-444a-b0aa-fe610e2287ac)

---

## **Step 2: Install Tekton Pipelines**

### **Why Tekton Pipelines?**
Tekton Pipelines allow us to define CI/CD workflows as Kubernetes-native resources. Unlike traditional CI/CD tools, Tekton runs entirely on Kubernetes, making it portable and scalable.

1. **Install Tekton Pipelines**  
   Run the following command to install Tekton Pipelines:
   ```bash
   kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
   ```

   - **Why?** This installs the Tekton components required to define and run pipelines.

2. **Verify Installation**  
   Check the Tekton components:
   ```bash
   kubectl get pods -n tekton-pipelines
   ```
   - **Why?** This ensures that Tekton is running correctly on your AKS cluster.

![Screenshot 2025-02-28 015841](https://github.com/user-attachments/assets/735f1857-8a61-4c02-bfa0-8d513de9858b)

---

## **Step 3: Configure the Tekton Pipeline**

### **What is a Tekton Pipeline?**
A Tekton Pipeline is a series of **tasks** that are executed in sequence or in parallel. Each task performs a specific part of the CI/CD workflow, such as cloning code, building an application, or deploying it.

---

### **Step 3.1: Install the `git-clone` Task**

The `git-clone` task is a reusable Tekton task that clones a Git repository into a workspace.  
```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.9/git-clone.yaml
```
- **Why?** We need to clone the application source code before building it.

---

### **Step 3.2: Create the Tekton YAML Files**

1. **Task: Build Step**  
   The `build-task.yaml` defines how to build the Node.js application:
   ```yaml
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
     name: build-task
   spec:
     workspaces:
       - name: shared-workspace
     steps:
       - name: build
         image: node:14
         workingDir: /workspace/shared-workspace
         script: |
           #!/bin/sh
           npm install
           npm run build
   ```
   - **Why?** This task installs dependencies and runs the build process.

2. **Pipeline: Connect Tasks**  
   The `pipeline.yaml` defines the CI/CD pipeline:
   ```yaml
   apiVersion: tekton.dev/v1beta1
   kind: Pipeline
   metadata:
     name: build-and-deploy-pipeline
   spec:
     workspaces:
       - name: pipeline-workspace
     tasks:
       - name: clone-repo
         taskRef:
           name: git-clone
         workspaces:
           - name: output
             workspace: pipeline-workspace
         params:
           - name: url
             value: https://github.com/heroku/node-js-sample.git
           - name: revision
             value: master
       - name: build-task
         taskRef:
           name: build-task
         workspaces:
           - name: shared-workspace
             workspace: pipeline-workspace
   ```
   - **Why?** The pipeline clones the repository, then builds the application.

---

### **Step 3.3: Apply the YAML Files**

Apply all the YAML files:
```bash
kubectl apply -f build-task.yaml
kubectl apply -f pipeline.yaml
```

---

## **Step 4: Run and Monitor the Pipeline**

1. **Start the Pipeline**  
   Trigger the pipeline manually:
   ```bash
   tkn pipeline start build-and-deploy-pipeline \
     --workspace name=pipeline-workspace,emptyDir=
   ```

2. **Monitor the Logs**  
   Follow the logs to verify the pipeline execution:
   ```bash
   tkn pipelinerun logs -f
   ```
   - **Why?** This ensures that the pipeline is functioning as expected.
   - 
![Screenshot 2025-02-28 024002](https://github.com/user-attachments/assets/a4f99762-951e-446e-97c8-dd01b4f10f04)

---

## **Conclusion**

This project demonstrates how to:
- Use Kubernetes (AKS) to run scalable CI/CD pipelines.
- Configure Tekton for cloud-native automation.
- Build reusable and modular CI/CD workflows.
