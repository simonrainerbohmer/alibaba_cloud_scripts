### Deploy an ECS Instance with 2 EIPs and 2 ENIs, and Run a Docker Image Using the SDK with Each EIP Bound to a Different Port on the ECS Instance

In this deployment, we set up an Elastic Compute Service (ECS) instance on Alibaba Cloud with 2 Elastic IPs (EIPs) and 2 Elastic Network Interfaces (ENIs). 
We then use Alibaba Cloud SDK to automate the creation and configuration of these resources. Additionally, we set up an Alibaba Cloud Container Registry (ACR) 
to store a Docker image, which we then pull and run on the ECS instance. 
Each EIP is bound to a different port on the ECS instance, allowing us to verify the deployment by accessing the running Docker containers via the assigned EIPs.

### Prerequisites

1. **Alibaba Cloud Account**:
   - Ensure you have an Alibaba Cloud account with the necessary permissions to create and manage resources.

2. **SDK Installation on Local Machine**:
    - Install the Alibaba Cloud Python SDK:
        ```sh
        pip install aliyun-python-sdk-core
        pip install aliyun-python-sdk-ecs
        pip install aliyun-python-sdk-vpc
        ```

3. **Access Keys**:
    - Obtain your Access Key ID and Secret from the Alibaba Cloud console.

4. **Container Registry Setup**:
    - Go to the Alibaba Cloud Console.
    - Navigate to the **Container Registry** service.
    - Create a new personal Container Registry instance (no need for enterprise edition).
    - Create a new repository within your ACR instance. Note down the repository name, namespace and password.

5. **Docker**:
    - Ensure Docker is installed on your local machine and the ECS instance (done after logging in to ECS).

6. **Python Environment**:
   - Ensure Python is installed on your local machine.

Make sure all prerequisites are met before proceeding with the setup and deployment steps.

### Step 1: Running the Script (Local Machine)

1. **Create `config.json` on your local machine**:
   - Ensure the `config.json` file in the same directory as the script `create_ecs_instance.py` contains the necessary environment variables. Adjust the values to match your Alibaba Cloud account details:
     ```json
     {
       "access_key_id": "your-access-key-id",
       "access_key_secret": "your-access-key-secret",
       "region_id": "eu-central-1",
       "vpc_id": "your-vpc-id",
       "security_group_id": "your-security-group-id",
       "vswitch_id": "your-vswitch-id"
     }
     ```

2. **Download the Script**:
   - Save the Python script `create_ecs_instance.py` to your local machine and in the same folder as `config.json`.

3. **Run the Script**:
   - Execute the script in your terminal:
     ```sh
     python create_ecs_instance.py
     ```

4. **Monitor Output**:
   - The script will output the progress of each step, including creating the ECS instance, attaching ENIs, allocating EIPs, and associating the EIPs with the instance and ENIs.

This sequence ensures that your ECS instance is set up with two EIPs and network interfaces, where EIP_1 is attached to ENI_1 respectively eth0 and EIP_2 is attached to ENI_2 respectively eth1.


### Step 2: Pull the Docker Image Locally (Local Machine)

1. **Log in to Docker Hub**:
   - Open a terminal on your local machine.
   - Log in to Docker Hub:
     ```sh
     docker login
     ```
   - Provide your Docker Hub username and password when prompted.

2. **Pull the Docker Image from Docker Hub**:
   - Pull the desired Docker image (e.g., Nginx) to your local machine (e.g., linux/amd64):
     ```sh
     docker pull --platform linux/amd64 nginx:latest
     ```

3. **Tag the Docker Image for ACR**:
   - Tag the pulled image with your ACR repository details:
     ```sh
     docker tag nginx:latest registry-intl.eu-central-1.aliyuncs.com/your-namespace/your-repository:latest
     ```

### Step 3: Push the Docker Image to ACR (Local Machine)

1. **Log in to ACR from Your Local Machine**:
   - Log in to your ACR instance using the Docker CLI:
     ```sh
     docker login --username=your-username@your-alibaba-cloud-id registry-intl.eu-central-1.aliyuncs.com
     ```
   - Replace `your-username@your-alibaba-cloud-id` with your Alibaba Cloud account username and `registry-intl.eu-central-1.aliyuncs.com` with your ACR registry endpoint.

2. **Push the Docker Image to ACR**:
   - Push the tagged image to your ACR repository:
     ```sh
     docker push registry-intl.eu-central-1.aliyuncs.com/your-namespace/your-repository:latest
     ```

### Step 4: Deploy the Docker Image on ECS (ECS Instance)

1. **Log in to ECS Instance**:
   - Log in to your ECS instance using the e.g., Alibaba Cloud console's Terminal Connect feature with temporary credentials.

2. **Install Docker on ECS**:
   - If Docker is not installed on your ECS instance, install it:
     ```sh
     sudo apt-get update
     sudo apt-get install -y docker.io
     sudo systemctl start docker
     ```

3. **Log in to ACR from ECS Instance**:
   - Log in to your ACR instance from the ECS instance using the Docker CLI (use password associated with pre-created ACR):
     ```sh
     sudo docker login --username=your-username@your-alibaba-cloud-id registry-intl.eu-central-1.aliyuncs.com
     ```

4. **Pull the Docker Image from ACR**:
   - Pull the Docker image from your ACR repository to the ECS instance:
     ```sh
     sudo docker pull registry-intl.eu-central-1.aliyuncs.com/your-namespace/your-repository:latest
     ```

5. **Run the pulled Docker image on the ECS instance with internal IP binding**:

    - Ensure the Docker container binds to the specific internal IPs associated with each ENI for proper traffic routing:
      ```sh
      sudo docker run -d -p 172.29.40.139:80:80 --name mynginx1 registry-intl.eu-central-1.aliyuncs.com/your-namespace/your-repository:latest
      sudo docker run -d -p 172.29.40.140:8080:80 --name mynginx2 registry-intl.eu-central-1.aliyuncs.com/your-namespace/your-repository:latest
      ```

    - Replace `172.29.40.139` and `172.29.40.140` with the internal IPs of your ENIs, and adjust the ports as needed.

### Step 5: Verify the Setup (ECS Instance)

1. **Check Docker Container Status**:
    - Verify that the Docker containers are running:
      ```sh
      sudo docker ps
      ```
    - You should see an output similar to this, indicating that the Nginx containers are running:
      ```sh
      CONTAINER ID      IMAGE                                                                            COMMAND                  CREATED                STATUS                PORTS                                       NAMES
      <container_id1>   registry-intl.eu-central-1.aliyuncs.com/your-namespace/your-repository:latest    "nginx -g 'daemon of…"   <time_since_created>   Up <time_since_up>    172.29.40.139:80->80/tcp                    mynginx1
      <container_id2>   registry-intl.eu-central-1.aliyuncs.com/your-namespace/your-repository:latest    "nginx -g 'daemon of…"   <time_since_created>   Up <time_since_up>    172.29.40.140:8080->80/tcp                  mynginx2
      ```

2. **Access the Running Nginx Servers (Local Machine)**:
    - Use the public static IPv4 addresses (EIPs) associated with the ECS instance to access the running Nginx servers (both are visible in the Alibaba Cloud console ECS section).
    - Open a web browser and navigate to the public IPv4 addresses:
      ```
      http://<Primary_EIP>
      http://<Secondary_EIP>:8080
      ```
    - You should see the Nginx welcome page indicating that the Nginx server is running successfully for both EIPs.

This sequence ensures that you have the image correctly tagged and pushed to ACR, and then pulled and run on your ECS instance. The setup is verified with each EIP bound to different ports, confirming that the configuration is working correctly.