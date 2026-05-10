# Azure DevOps Deployment Guide

This document provides a clear and detailed blueprint to automate the deployment of the TravelMemory application using Azure DevOps. It strictly follows the manual steps outlined in the `backend.md` and `frontend.md` documentation.

Since the project was successfully run manually, this guide assumes Node.js and NPM are already installed on your Azure Virtual Machine.

---

## Step 1: Set up the Azure DevOps Environment (Virtual Machine Agent)

Instead of SSH-ing manually into the server each time, we will install an Azure Agent on your VM. This allows Azure DevOps to run the deployment commands directly on your machine.

1. Go to **Pipelines** > **Environments** in your Azure DevOps project.
2. Click **New environment**, and name it exactly `TravelMemory-VM`.
3. Select **Virtual machines** as the resource type and click **Next**.
4. Select **Linux** as the Operating System.
5. Azure DevOps will provide a **Registration script** (a long `bash` command).
6. Copy that script, SSH into your Azure VM, paste it, and press **Enter** to run it. 
   *(This connects your VM to Azure DevOps so it can receive deployment tasks).*

---

## Step 2: Configure Environment Variables in Azure DevOps

The application requires `.env` files for both the frontend and backend. We will store these values securely in Azure DevOps so the pipeline can inject them during deployment.

1. Go to **Pipelines** > **Library** in Azure DevOps.
2. Click **+ Variable group** and name it `TravelMemory-Vars`.
3. Under the **Variables** section, click **+ Add** and add the following variables:
   * **Name:** `MONGO_URI` | **Value:** `Your MongoDB Connection String`
   * **Name:** `BACKEND_PORT` | **Value:** `3001`
   * **Name:** `REACT_APP_BACKEND_URL` | **Value:** `http://<YOUR_VM_PUBLIC_IP>:3001` *(Ensure there is no trailing slash)*
4. Toggle **Allow access to all pipelines** in the security settings for this group.
5. Click **Save**.

---

## Step 3: Open the Required Ports in Azure Portal

For the application to be accessible to the public, the appropriate network ports must be open. The frontend will be served on Port 80, and the backend on Port 3001.

1. Go to your Virtual Machine in the Azure Portal.
2. Go to **Networking** on the left menu.
3. Add an inbound port rule for **Port 80** (HTTP).
4. Add another inbound port rule for **Port 3001** (Backend API).

---

## Step 4: Create the Pipeline (`azure-pipelines.yml`)

The pipeline will execute the exact commands from your documentation.

> **Important Note:** In the manual steps, commands like `node index.js` and `npm start` run in the foreground. If an automated pipeline runs these commands exactly as-is, the pipeline will hang forever waiting for the server to stop. To fix this, the commands in the pipeline are wrapped in `nohup ... &` to run them in the background. Additionally, `sudo PORT=80` is prefixed to the frontend command to fulfill the requirement of running it on Port 80 instead of the default 3000.

Create an `azure-pipelines.yml` file in the root of your repository and paste the following configuration:

```yaml
trigger:
  - main

# Import the variables we created in Step 2
variables:
  - group: TravelMemory-Vars

jobs:
  - deployment: DeployToVM
    environment: 
      name: 'TravelMemory-VM'
      resourceType: VirtualMachine
    strategy:
      runOnce:
        deploy:
          steps:
            # Step 1: Cloning the Repository
            # Azure DevOps automatically clones your repo to the VM using this checkout step
            - checkout: self  
            
            # Clean up: Kill any previously running Node servers before starting new ones
            - script: |
                sudo pkill node || true
              displayName: 'Stop previous servers'

            # ---------------------------------------------
            # BACKEND DEPLOYMENT (Based on backend.md)
            # ---------------------------------------------
            - script: |
                # Step 2: Navigate to the Backend Folder
                cd $(System.DefaultWorkingDirectory)/backend
                
                # Step 3: Create the .env File
                echo "MONGO_URI=$(MONGO_URI)" > .env
                echo "PORT=$(BACKEND_PORT)" >> .env
                
                # Step 4: Install Necessary Packages
                npm install
                
                # Step 5: Run the Backend Application
                nohup node index.js > backend.log 2>&1 &
              displayName: 'Deploy Backend'

            # ---------------------------------------------
            # FRONTEND DEPLOYMENT (Based on frontend.md)
            # ---------------------------------------------
            - script: |
                # Step 2: Navigate to Your Frontend Directory
                cd $(System.DefaultWorkingDirectory)/frontend
                
                # Step 3: Create the .env File
                echo "REACT_APP_BACKEND_URL=$(REACT_APP_BACKEND_URL)" > .env
                
                # Step 4: Install Required Packages
                npm install
                
                # Step 5: Run Your Application
                # (Running with PORT=80 to meet your requirement)
                sudo PORT=80 nohup npm start > frontend.log 2>&1 &
              displayName: 'Deploy Frontend'
```

---

## Step 6: Run and Verify

1. **Commit and Push:** Save the `azure-pipelines.yml` file and push it to your repository.
2. **Trigger Pipeline:** In Azure DevOps, the pipeline will trigger automatically upon detecting the new commit in the `main` branch.
3. **Verify Deployment:** Once the pipeline succeeds, open your web browser and navigate to `http://<YOUR_VM_PUBLIC_IP>`. 
   
The frontend should be running on Port 80 and properly communicating with your backend on Port 3001!
