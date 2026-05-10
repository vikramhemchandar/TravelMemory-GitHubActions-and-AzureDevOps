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

The pipeline will execute the exact commands from your documentation, but adjusted with advanced configurations to survive the background process cleanup of the Azure DevOps Agent.

Create an `azure-pipelines.yml` file in the root of your repository and paste the following configuration:

```yaml
trigger:
  - main

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
            - checkout: self  
            
            # This explicitly injects Node.js & NPM into the pipeline agent's PATH
            - task: NodeTool@0
              inputs:
                versionSpec: '18.x'
              displayName: 'Install Node.js'
            
            # Clean up: Kill ONLY our app servers, not the pipeline agent!
            - script: |
                sudo pkill -f "node index.js" || true
                sudo pkill -f "react-scripts" || true
              displayName: 'Stop previous servers'

            # ---------------------------------------------
            # BACKEND DEPLOYMENT
            # ---------------------------------------------
            - script: |
                cd $(System.DefaultWorkingDirectory)/backend
                
                echo "MONGO_URI=$(MONGO_URI)" > .env
                echo "PORT=$(BACKEND_PORT)" >> .env
                
                npm install
                
                # We use NodeTool@0 so 'node' is found.
                # Added sudo env PATH=$PATH $(which node) so Azure DevOps can't kill it!
                sudo env PATH=$PATH nohup $(which node) index.js </dev/null > backend.log 2>&1 &
              displayName: 'Deploy Backend'

            # ---------------------------------------------
            # FRONTEND DEPLOYMENT
            # ---------------------------------------------
            - script: |
                cd $(System.DefaultWorkingDirectory)/frontend
      
                echo "REACT_APP_BACKEND_URL=$(REACT_APP_BACKEND_URL)" > .env
                
                npm install
                
                # We use env PATH=$PATH and $(which npm) so sudo doesn't lose track of NPM
                sudo BROWSER=none PORT=80 env PATH=$PATH nohup $(which npm) start </dev/null > frontend.log 2>&1 &
              displayName: 'Deploy Frontend'

            # ---------------------------------------------
            # FIX PERMISSIONS FOR THE NEXT RUN
            # ---------------------------------------------
            - script: |
                sudo chown -R $USER:$USER $(System.DefaultWorkingDirectory)
              displayName: 'Fix Folder Permissions for Next Run'
              condition: always()
```

### Detailed Explanation of Pipeline Steps

1. **Trigger and Variables**
   - `trigger: - main`: Automatically runs the pipeline whenever code is pushed to the `main` branch.
   - `variables: - group: TravelMemory-Vars`: Injects the environment variables (like `MONGO_URI` and `PORT`) from the Azure DevOps Library into the pipeline so they can be securely written to `.env` files.

2. **Environment & Strategy**
   - `environment: name: 'TravelMemory-VM'`: Tells Azure DevOps to run this specific job directly on the Virtual Machine where you installed the agent.
   - `checkout: self`: Automatically pulls the latest code from your repository into the VM's working directory (`$(System.DefaultWorkingDirectory)`).

3. **Install Node.js Task (`NodeTool@0`)**
   - The Azure DevOps Agent runs as a background service and does not have access to your personal `.bashrc` profile, meaning it cannot natively find commands like `node` or `npm`.
   - This step forces Azure DevOps to explicitly download Node.js 18.x and inject it directly into the pipeline's system `PATH`.

4. **Stop Previous Servers Script**
   - Before deploying new code, we must kill the old running servers to prevent "Port already in use" errors.
   - We use `sudo pkill -f "node index.js"` instead of a broad `pkill node` because the Azure DevOps Agent itself relies on Node.js! If we kill all `node` processes, we kill the agent and the pipeline fails.

5. **Deploy Backend Script**
   - Navigates to the backend folder and securely writes the variables from Azure DevOps to the local `.env` file.
   - Runs `npm install` to download dependencies.
   - `sudo env PATH=$PATH nohup $(which node) index.js </dev/null > backend.log 2>&1 &`: 
     - **`sudo env PATH=$PATH $(which node)`**: Runs the backend with Admin privileges while forcing `sudo` to remember where Node is installed.
     - **`nohup ... &`**: Runs the process in the background so the pipeline can move on.
     - **`</dev/null`**: Completely detaches the process from Azure DevOps inputs. Without this, Azure DevOps will aggressively kill the backend process at the end of the job (Process Cleanup feature).

6. **Deploy Frontend Script**
   - Navigates to the frontend folder and creates its `.env` file.
   - Runs `npm install`.
   - `sudo BROWSER=none PORT=80 env PATH=$PATH nohup $(which npm) start </dev/null > frontend.log 2>&1 &`:
     - **`PORT=80`**: Forces the React development server to run on the standard HTTP Port 80 instead of the default 3000.
     - **`BROWSER=none`**: Prevents React from trying to launch an interactive web browser window on the VM, which would cause the CI/CD pipeline to hang.

7. **Fix Folder Permissions for Next Run**
   - Because we ran the frontend using `sudo npm start`, React created cache files (like babel and eslint caches) inside `node_modules` that are strictly owned by the `root` user.
   - When the next pipeline triggers, the `checkout: self` step runs as the normal `azureuser` and will fail with a **Permission Denied** error when trying to clean the workspace.
   - This `condition: always()` script forces the ownership of the files back to the normal user at the very end of the pipeline, ensuring the workspace is perfectly clean for the next deployment.

---

## Step 5: Run and Verify

1. **Commit and Push:** Save the `azure-pipelines.yml` file and push it to your repository.
2. **Trigger Pipeline:** In Azure DevOps, the pipeline will trigger automatically upon detecting the new commit in the `main` branch.
3. **Verify Deployment:** Once the pipeline succeeds, open your web browser and navigate to `http://<YOUR_VM_PUBLIC_IP>`. You do not need to append the port number since the frontend runs on Port 80.
   
The frontend should be successfully communicating with your backend on Port 3001!

---

## Common Troubleshooting

### 1. "The agent has received a shutdown signal"
The Azure DevOps Agent runs on Node.js under the hood. If you run a generic cleanup command like `sudo pkill node`, it kills the Azure Pipelines Agent, causing the deployment to fail instantly. Always use specific names like `sudo pkill -f "node index.js"`.

### 2. "Command not found: node" or "npm"
The Azure DevOps background service does not read your `.bashrc` profile, so it doesn't know where Node is installed. To fix this, always include the `NodeTool@0` task in your pipeline to inject Node into the PATH, and use `$(which node)` when using `sudo`.

### 3. Backend starts but immediately crashes (Missing from netstat)
If the backend logs show it started, but you cannot connect on port 3001, it crashed after startup. Since Azure assigns new Public IPs when VMs are rebooted, **MongoDB Atlas** will block the new IP. Ensure your VM's public IP is whitelisted in MongoDB Atlas under Network Access.

### 4. Permission Denied during "Checkout" Step
Running `sudo npm start` creates cache files owned by the `root` user in your workspace. When the next pipeline attempts to pull code, the normal agent user cannot clean up these files. We solve this by adding a `condition: always()` step at the end of the pipeline that runs `sudo chown -R $USER:$USER $(System.DefaultWorkingDirectory)` to reset ownership.

