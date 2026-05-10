# Deploy the React Frontend

Follow these steps to deploy the React frontend.

### Step 1: SSH to Your EC2 Instance
To connect to your EC2 instance, you need to use the SSH command in your terminal. Make sure you have your key pair (`.pem` file) and the public IPv4 address of your instance. The command format is as follows:
```bash
ssh -i /path/to/your-key.pem ubuntu@EC2_PUBLIC_IP
```

### Step 2: Navigate to Your Frontend Directory
Once connected, navigate to the directory where your frontend application is located using the following command:
```bash
cd TravelMemory/frontend
```

### Step 3: Create the `.env` File
In your frontend directory, create a `.env` file to store environment variables. This file will contain the backend URL that your React app will use to connect to the server.
```bash
echo "REACT_APP_BACKEND_URL=http://EC2_PUBLIC_IP:3001" > .env
```

### Step 4: Install Required Packages
Now that your environment variables are set, you need to install the required packages for your application. Run the following command:
```bash
npm install
```

### Step 5: Run Your Application
After the packages are installed, you can run your application using the command below. This will start your React application on port `3000`.
```bash
npm start
```

### Step 6: Access Your Application
You can access your deployed React application by navigating to `http://PUBLIC_IP:3000` in your web browser. 

> **Note:** Make sure your security group settings allow inbound traffic on port `3000`.
