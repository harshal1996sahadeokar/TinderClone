# TinderClone
Here's a detailed step-by-step guide for developing a dating app clone using **AWS**, **Terraform**, **GitLab**, and **Docker**:

### **Step 1: Project Setup**

1. **Initialize your repository**:
   - Create a **GitLab** repository for your project. Push the initial structure of your dating app project to this repo. If you're using GitLab, run the following commands to set up the Git repository:
     ```bash
     git init
     git remote add origin https://gitlab.com/<your-username>/<repo-name>.git
     git add .
     git commit -m "Initial commit"
     git push -u origin main
     ```

2. **Define Project Structure**:
   - Divide the application into two tiers:
     - **Frontend**: React/HTML5-based.
     - **Backend**: Node.js, Python, or Java-based API using Express, Flask, or Spring.
   - The `docker-compose.yml` will manage both tiers.

### **Step 2: Dockerizing the Application**

1. **Dockerfile for Backend**:
   - Create a `Dockerfile` for the backend (e.g., for a Node.js API):
     ```dockerfile
     # Use Node.js as the base image
     FROM node:14-alpine
     WORKDIR /usr/src/app
     COPY package*.json ./
     RUN npm install
     COPY . .
     EXPOSE 3000
     CMD ["npm", "start"]
     ```

2. **Dockerfile for Frontend**:
   - Similarly, for a React frontend:
     ```dockerfile
     FROM node:14-alpine
     WORKDIR /app
     COPY package*.json ./
     RUN npm install
     COPY . .
     EXPOSE 3000
     CMD ["npm", "start"]
     ```

3. **Docker Compose Setup**:
   - A `docker-compose.yml` file to connect both frontend and backend:
     ```yaml
     version: '3'
     services:
       backend:
         build: ./backend
         ports:
           - "3001:3001"
         depends_on:
           - db
       frontend:
         build: ./frontend
         ports:
           - "3000:3000"
         depends_on:
           - backend
       db:
         image: mysql:5.7
         environment:
           MYSQL_ROOT_PASSWORD: rootpassword
           MYSQL_DATABASE: dating_app
           MYSQL_USER: user
           MYSQL_PASSWORD: password
         ports:
           - "3306:3306"
     ```

4. **Running Docker Compose**:
   - Use the following command to build and run both the frontend and backend services:
     ```bash
     docker-compose up --build
     ```

### **Step 3: AWS Integration**

1. **Setting up an EC2 instance**:
   - Use **Terraform** to automate the deployment of your EC2 instances where Docker will run.
   - Example Terraform script:
     ```hcl
     provider "aws" {
       region = "ap-south-1"
     }

     resource "aws_instance" "app" {
       ami           = "ami-0c55b159cbfafe1f0" # Ubuntu 20.04 AMI
       instance_type = "t2.micro"
       key_name      = "your-key-pair"

       provisioner "remote-exec" {
         inline = [
           "sudo apt-get update -y",
           "sudo apt-get install docker.io -y",
           "sudo systemctl start docker",
           "sudo systemctl enable docker"
         ]
       }
     }

     output "ec2_public_ip" {
       value = aws_instance.app.public_ip
     }
     ```

2. **Configuring AWS ECR (Elastic Container Registry)**:
   - Store Docker images in AWS **ECR**. Push your Docker images to ECR using the following commands:
     ```bash
     aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
     docker tag <image_name>:latest <aws_account_id>.dkr.ecr.<region>.amazonaws.com/<repo_name>:latest
     docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/<repo_name>:latest
     ```

3. **Deploy to ECS (Elastic Container Service)**:
   - Use **Terraform** to deploy Docker containers on ECS with Fargate, for serverless container management:
     - **ECS Cluster** creation:
       ```hcl
       resource "aws_ecs_cluster" "dating_app_cluster" {
         name = "dating-app-cluster"
       }
       ```
     - **ECS Task Definition** to run the container:
       ```hcl
       resource "aws_ecs_task_definition" "dating_app" {
         family                   = "dating-app-task"
         network_mode             = "awsvpc"
         requires_compatibilities = ["FARGATE"]
         memory                   = "512"
         cpu                      = "256"

         container_definitions = jsonencode([
           {
             name      = "dating-app-backend",
             image     = "<ecr_repo_url>:latest",
             cpu       = 256,
             memory    = 512,
             essential = true,
             portMappings = [
               {
                 containerPort = 3000
                 hostPort      = 3000
               }
             ]
           }
         ])
       }
       ```

### **Step 4: CI/CD with GitLab**

1. **GitLab CI/CD Pipeline**:
   - Create a `.gitlab-ci.yml` file to automate the deployment.
   - Here’s an example to build the Docker image and push it to ECR, then deploy it to ECS:
     ```yaml
     stages:
       - build
       - deploy

     build_job:
       stage: build
       script:
         - docker build -t dating-app-backend .
         - docker tag dating-app-backend:latest <aws_account_id>.dkr.ecr.<region>.amazonaws.com/dating-app-backend:latest
         - $(aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com)
         - docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/dating-app-backend:latest

     deploy_job:
       stage: deploy
       script:
         - aws ecs update-service --cluster dating-app-cluster --service dating-app-service --force-new-deployment
     ```

### **Step 5: Monitoring and Logging**

1. **Set up CloudWatch**:
   - For monitoring your ECS containers, configure **CloudWatch Logs** to monitor the logs of your application.
   - Set up **CloudWatch Alarms** to notify you of CPU/memory threshold breaches in ECS.

### **Conclusion**

This project setup gives you full control over the infrastructure and deployment of a 2-tier dating app using AWS services, automated with **Terraform** and **GitLab CI/CD**, and containerized with **Docker** and **Docker Compose**.

By following these steps, you'll get a hands-on understanding of how to implement a cloud-native application using industry-standard tools.
