# **CI/CD Pipeline with Jenkins Docker Agents for AWS ECS**

This project demonstrates an end-to-end CI/CD pipeline using Jenkins to build, push, and deploy a containerized application to AWS. The pipeline leverages Docker containers as agents for each build stage, ensuring a clean and consistent environment.

## **Core concepts to take note of:**

-   **Jenkins Docker Agents:** Each pipeline stage runs inside a dedicated Docker container, eliminating the need to pre-install build tools on the Jenkins server.
    
-   **Automated Deployment:** The pipeline automatically builds a Docker image, pushes it to ECR, and updates an ECS service to deploy the latest version.
    
-   **Secure Secrets Management:** Database credentials and other sensitive data are stored in AWS Parameter Store instead of being hardcoded.

   ##  Architecture Overview

| Component | Role |
|-----------|------|
| **Jenkins (EC2)** | Manages and administrative tasks. It is the central automation server where administrative activities are performed. I configured with the Docker Pipeline Plugin to allow `agent docker {}` directives in the Jenkinsfile.|
| **Docker** | For containerizing the application and serving as a Jenkins build agent, provideing an ephemeral, isolated environment for each pipeline stage. The host’s Docker socket is mounted inside the container to allow building and pushing images.|
| **IAM** | For managing user and service permissions. The ECS Task Role was created for accessing S3 and Parameter Store, while the ECS Task Execution Role for pulling images from ECR |
| **AWS ECR** |A private Docker registry for storing application images |
| **ECS** | A container orchestration service using the Fargate serverless compute engine. The reason for Fargate is that we want containers that run without managing the underlying servers for quick scaling and reduced operational overhead |
| **S3** | The centralized file store for storing static files/user uploads. |
| **RDS** | Database service for the MySQL database. |
| **Parameter Store** | For storing configuration data and secrets. It preventing our credentials from being hardcoded into source code |
| **CloudWatch** | For monitoring and log management via Container Insights.. |
---

## **Implementation  and Workflow**

1.  **AWS Infrastructure Setup:**
    -   Create an IAM user for Jenkins with programmatic access. The Access Key and Secret Access Key that are gotten from IAM can then be securely stored in Jenkins using the AWS Credentials Plugin.
        
    -   Set up a private ECR repository to store our application's Docker images. Authentication with this repository is handled by IAM roles and the AWS credentials in Jenkins.
        
    -   Create an S3 bucket for file uploads, as it would be the object storage for uploaded files from the application
        
    -   Provision an RDS (MySQL) database with a restricted security group. The security group rules would be configured in such a way that the only allowed source was the security group of the ECS cluster, ensuring the database is not exposed to the public internet.
        
    -   Store database credentials in AWS Parameter Store in order to have a secure credential management. Database connection details (host, name, username, password) are not to be hardcoded in the application. Instead, they were securely stored as `SecureString` parameters in AWS Parameter Store.
        
    -   Create an ECS Cluster, Task Definition, and Service with an Application Load Balancer. The ECS cluster that was created using the **Fargate** serverless launch type, eliminates the need to manage EC2 instances. The **Task Role** was created to grant the container permissions to interact with other AWS services. This role was given `S3FullAccess` and `SSMReadOnlyAccess` policies, allowing the application to upload files to S3 and fetch database credentials from Parameter Store.  A **Task Execution Role** was created to allow the ECS service to pull images from ECR. This role comes with the necessary `AmazonECSTaskExecutionRolePolicy`.
        
2.  **Jenkins Configuration:**
    
    -   Install required plugins: Docker Pipeline, AWS SDK All, AWS Credentials, Amazon ECR, and Pipeline: AWS Steps.
        
    -   Store your AWS credentials in Jenkins.
        
    -   Create a Jenkins Pipeline job linked to your GitHub repository via webhooks.
        
3.  **Run the Pipeline:**
Before we run the pipeline, lets walkthrough the deployment file:
#### Setup Script :
```bash

        stage('Build Docker Image') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'  // Mount Docker socket
                }
            }
            steps {
                script {
                    echo 'Building Docker Image from Dockerfile...'
                    sh 'mkdir -p /tmp/.docker'  // Ensure the directory exists
                    dockerImage = docker.build(repoUri + ":$BUILD_NUMBER")
                }
            }
        }

        stage('Push Docker Image to ECR') {
            agent {
                docker {
                    image 'docker:latest'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'  // Mount Docker socket
                }
            }
            steps {
                script {
                    echo "Pushing Docker Image to ECR..."
                    docker.withRegistry(repoRegistryUrl, registryCreds) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            agent {
                docker {
                    image 'amazon/aws-cli:latest'  // Use a pre-built AWS CLI Docker image for ECS deployment
                    args '-v /var/run/docker.sock:/var/run/docker.sock --entrypoint=""'  // Optional if needed by AWS CLI
                }
            }
            steps {
                script {
                    echo "Deploying Image to ECS..."
                    withAWS(credentials: 'awscreds', region: "${region}") {
                        sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
                    }
                }
            }
        } 
    }
}
```
In the setup script, certain keywords are to be nored:

-   **Agent Configuration (`agent none`):** By setting `agent none` globally, it simply means each stage can specify its own agent, allowing for different tools and environments per task . For example, `docker:latest` for the Docker stages, `aws-cli` for the deployment stage
-   **`DOCKER_CONFIG` Environment Variable:** A temporary directory is specified to give the Docker agent a workspace to mount volumes, which is a required step for using Docker agents with Jenkins. 
-   **Stage: Docker Test:** This stage's purpose is to verify that the Docker agent container can successfully communicate with the host Jenkins server's Docker daemon. Here, one point is crucial. One must set Docker socket permissions (`chmod 666 /var/run/docker.sock`) so Docker agents could build and push images.
-   **Stage: Build & Push:** The `docker.build()` and `docker.withRegistry()` methods from the Docker Pipeline Plugin were used, providing a cleaner, more robust alternative to raw shell commands. The Docker Pipeline plugin proved to be very helpful in order to use these methods, rather than commands. The images are tagged with the Jenkins `BUILD_NUMBER` and a `latest` tag, providing clear versioning and traceability.
-   **Stage: Deploy to ECS:** This stage uses an `aws-cli` Docker image as its agent. One detail that stands out is `-entrypoint=""` argument**.** The default `aws-cli` image has a default command (`ENTRYPOINT`) that runs on container startup. Now, to ensure our custom AWS commands within the Jenkinsfile are executed, the `-entrypoint=""` argument was used to override the default entrypoint. This is a subtle but critical detail for successful deployment.

**Going ahead to run the pipeline:**
    -   Push a code change to the main branch of this repository.
    -   Jenkins will automatically trigger the pipeline, executing the build, push, and deploy stages.
    -   Monitor the pipeline's progress and logs in the Jenkins UI.

## **Verification for deployment**

1.  **Application Accessibility:** Once the pipeline completed, the application is made accessible via the public DNS of the Application Load Balancer, confirming successful service deployment.
2.  **Service Functionality:**
    -   The application's form was tested to upload a file and submit data.
    -   The form submission was successful, confirming that the application could fetch database credentials from Parameter Store and write data to the RDS database.

## **Outcomes and Results**

By the end of the project, the CI/CD pipeline was successfully established, where every code change in GitHub automatically triggered a full build-push-deploy cycle without any manual intervention. This process was highly reliable and consistent, as every build was executed within a clean and isolated container environment, guaranteeing that dependencies were managed effectively and reliably.
