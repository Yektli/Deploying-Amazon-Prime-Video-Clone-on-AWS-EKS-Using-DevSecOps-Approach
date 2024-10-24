# Deploy Amazon Prime Clone Application AWS using DevSecOps Approach


### Project Summary:

This project involves the deployment of a Prime Video clone using a full-stack serverless architecture on AWS. It utilizes several AWS services to create a scalable, high-performance streaming platform that mimics the functionality of Amazon's Prime Video.

The architecture includes AWS Amplify for seamless front-end hosting and CI/CD integration, and AWS Lambda functions for handling backend logic and processing. DynamoDB is used as a NoSQL database to manage user data and video metadata efficiently, while Amazon S3 is utilized for scalable storage of video files and assets. AWS CloudFront is integrated to deliver content globally with low latency, ensuring a smooth streaming experience.

Authentication and user management are handled by Amazon Cognito, providing secure sign-in and user data management. The deployment setup uses Infrastructure as Code (IaC) principles, implemented with AWS CloudFormation templates, to automate resource provisioning, ensuring consistency and reducing manual configuration.

This project showcases a robust deployment approach for a streaming service by utilizing the capabilities of serverless computing, scalable storage, and global content delivery on AWS.

### Project Architecture:

![Picture1](https://github.com/user-attachments/assets/c8e43dc4-0eb3-4ba1-b435-762e336762b8)

### Development summarized in steps
- Code Checkout & Analysis: Jenkins fetches the source code from GitHub and performs SonarQube analysis.
- Security Scans: OWASP and Trivy scan the codebase for vulnerabilities.
- Dockerization: Jenkins builds, tags, and pushes the Docker image to DockerHub.
- Deployment: The Docker image is deployed to AWS EC2 containers and monitored using Prometheus.
- Monitoring Setup: Prometheus and Grafana are configured to collect and visualize performance metrics.
- ArgoCD Deployment: ArgoCD automates the deployment of the application to Kubernetes, allowing for automatic sync and self-healing.
- Final Cleanup: AWS EC2 instances are terminated once the deployment process is completed.

### Key Technologies and Tools:
- Jenkins: Used as the CI/CD pipeline tool, automating each stage of deployment.
- SonarQube: For static code analysis to ensure code quality and security.
- OWASP Dependency Check: Scans for vulnerabilities in project dependencies.
- Trivy: A tool for scanning file systems for vulnerabilities.
- Docker: Used to containerize the application, build Docker images, and push them to DockerHub.
- Prometheus & Node Exporter: For monitoring the application and infrastructure.
- Grafana: Provides a dashboard for visualizing metrics.
- ArgoCD: Manages continuous deployment to Kubernetes.
- Kubernetes: Orchestrates the deployed containers.
- AWS EC2: Hosts instances for the application and its supporting services.
 
# Jenkins Complete pipeline
```
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage ("clean workspace") {
            steps {
                cleanWs()
            }
        }
        stage ("Git checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/yeshwanthlm/Prime-Video-Clone-Deployment.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=amazon-prime \
                    -Dsonar.projectKey=amazon-prime '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage ("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }
        stage ("Build Docker Image") {
            steps {
                sh "docker build -t amazon-prime ."
            }
        }
        stage ("Tag & Push to DockerHub") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker tag amazon-prime amonkincloud/amazon-prime:latest "
                        sh "docker push amonkincloud/amazon-prime:latest "
                    }
                }
            }
        }
        stage('Docker Scout Image') {
            steps {
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh 'docker-scout quickview amonkincloud/amazon-prime:latest'
                       sh 'docker-scout cves amonkincloud/amazon-prime:latest'
                       sh 'docker-scout recommendations amonkincloud/amazon-prime:latest'
                   }
                }
            }
        }
        stage ("Deploy to Conatiner") {
            steps {
                sh 'docker run -d --name amazon-prime -p 3000:3000 amonkincloud/amazon-prime:latest'
            }
        }
    }
    post {
    always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: """
                <html>
                <body>
                    <div style="background-color: #FFA07A; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Project: ${env.JOB_NAME}</p>
                    </div>
                    <div style="background-color: #90EE90; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">Build Number: ${env.BUILD_NUMBER}</p>
                    </div>
                    <div style="background-color: #87CEEB; padding: 10px; margin-bottom: 10px;">
                        <p style="color: white; font-weight: bold;">URL: ${env.BUILD_URL}</p>
                    </div>
                </body>
                </html>
            """,
            to: 'provide_your_Email_id_here',
            mimeType: 'text/html',
            attachmentsPattern: 'trivy.txt'
        }
    }
}

