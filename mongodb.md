# Configure the Mongo Database

Prepare your MongoDB database for application deployment.

### Step 1: Create a Cluster on MongoDB Atlas
1. Sign in to your MongoDB Atlas account. If you don’t have an account, create one at [MongoDB Atlas](https://www.mongodb.com/cloud/atlas).
2. After logging in, click on the **Build a Cluster** button.
3. Choose **AWS** as the cloud provider and select a region closest to you.
4. Select the free tier option (**M0**).
5. Click on **Create Cluster** and wait for it to be provisioned (this may take a few minutes).

### Step 2: Allow Access from Anywhere
1. Navigate to **Database Access** in the left sidebar.
2. Click on **Add New Database User**.
3. Enter a username and password. 
   > **Note:** Make sure to store these securely. The password should not have special characters, just alphanumeric.
4. Under **Database User Privileges**, select **Read and write to any database**.
5. Click **Add User**. 

Next, go to **Network Access** on the left sidebar:
1. Click **Add IP Address**.
2. Select **Allow Access from Anywhere** or add your specific IP address.
3. Confirm and save changes.

### Step 3: Create a Database Using MongoDB Compass
1. Download MongoDB Compass from the [official website](https://www.mongodb.com/try/download/compass).
2. Follow the installation instructions for your operating system.
3. Open MongoDB Compass and, in the connection window, input your connection URI. 
   
   **Example of a connection URI:** 
   ```text
   mongodb+srv://<username>:<password>@cluster0.mongodb.net/travelmemory?retryWrites=true&w=majority
   ```
4. Replace `<username>` and `<password>` with the credentials you created earlier.
5. Click **Connect**. 
6. Once connected, click on **Create Database**, name it `travelmemory`, and create a collection (e.g., `trips`).

### Step 4: Save Your Cluster Connection URI
Make sure to store your MongoDB connection URI in a secure place as you will need it for future configurations. The URI allows you to connect to your database from various applications.

**Sample URI:**
```text
mongodb+srv://myUser:myPassword@cluster0.mongodb.net/travelmemory?retryWrites=true&w=majority
```