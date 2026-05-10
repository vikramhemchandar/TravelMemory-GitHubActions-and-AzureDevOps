# Deploy the Backend Application

Let's deploy the backend application first.

### Step 1: Cloning the Repository
First, you need to clone the GitHub repository to your EC2 instance. Use the following command in your terminal:
```bash
git clone https://github.com/UnpredictablePrashant/TravelMemory.git
```

### Step 2: Navigate to the Backend Folder
Once cloned, navigate into the backend folder using the command below:
```bash
cd TravelMemory/backend
```

### Step 3: Create the `.env` File
Create a `.env` file in the backend directory to store your environment variables. Use the command below to create and edit the file:
```bash
nano .env
```
Add the following content to your `.env` file, replacing `ENTER_YOUR_URL` with your MongoDB URI from Atlas:
```env
MONGO_URI='ENTER_YOUR_URL'
PORT=3001
```

### Step 4: Install Necessary Packages
Next, you need to install the required packages. Run the following command to install the dependencies defined in `package.json`:
```bash
npm install
```

### Step 5: Run the Backend Application
Start the backend server using the command below. This will run the server on port `3001`:
```bash
node index.js
```

### Step 6: Test the API Endpoint
To verify that your backend is running successfully, navigate to your IP address followed by `:3001/hello` in your web browser. 

**Example:**
```text
http://YOUR_IP:3001/hello
```
You should see the message `Hello World`.