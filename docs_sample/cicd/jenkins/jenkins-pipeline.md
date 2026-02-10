# Jenkins Pipeline

## 🔧 Jenkins Overview

**Jenkins** là leading open-source automation server cho CI/CD.

**Key Features:**
- ✅ 1800+ plugins
- ✅ Pipeline as Code (Jenkinsfile)
- ✅ Distributed builds
- ✅ Extensible architecture
- ✅ Community support

**Use Cases:**
- CI/CD pipelines
- Automated testing
- Deployment automation
- Scheduled jobs
- Infrastructure automation

## 📦 Installation

### **Method 1: Docker (Recommended for Development)**

```bash
# Run Jenkins container
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts

# Get initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword

# Access: http://localhost:8080
```

### **Method 2: Docker Compose**

```yaml
# docker-compose.yml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins
    privileged: true
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false

volumes:
  jenkins_home:
```

```bash
docker-compose up -d
```

### **Method 3: Kubernetes**

```bash
# Add Helm repo
helm repo add jenkinsci https://charts.jenkins.io
helm repo update

# Install Jenkins
helm install jenkins jenkinsci/jenkins \
  --namespace jenkins \
  --create-namespace \
  --set controller.serviceType=LoadBalancer

# Get admin password
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins -- \
  /bin/cat /run/secrets/additional/chart-admin-password
```

### **Method 4: Linux (Ubuntu/Debian)**

```bash
# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
  sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Java
sudo apt update
sudo apt install openjdk-11-jdk -y

# Install Jenkins
sudo apt install jenkins -y

# Start Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Get initial password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## 🎨 Jenkins Configuration

### **Initial Setup**

1. **Access Jenkins**: http://localhost:8080
2. **Enter initial admin password**
3. **Install suggested plugins**
4. **Create first admin user**
5. **Configure Jenkins URL**

### **Essential Plugins**

```groovy
// Manage Jenkins → Plugin Manager → Available

// Source Control
- Git plugin
- GitHub plugin
- GitLab plugin

// Build Tools
- Maven Integration
- Gradle
- NodeJS

// Docker
- Docker plugin
- Docker Pipeline
- Docker Build Step

// Cloud
- Kubernetes
- AWS Steps
- Azure VM Agents

// Notifications
- Email Extension
- Slack Notification
- Microsoft Teams

// Testing
- JUnit
- Code Coverage API
- HTML Publisher

// Security
- OWASP Dependency-Check
- SonarQube Scanner

// Pipeline
- Pipeline
- Pipeline: Stage View
- Blue Ocean
```

### **Global Tool Configuration**

```
Manage Jenkins → Global Tool Configuration

JDK:
  Name: JDK11
  JAVA_HOME: /usr/lib/jvm/java-11-openjdk-amd64

Maven:
  Name: Maven3
  Version: 3.8.6

Git:
  Name: Default
  Path: git

Docker:
  Name: Docker
  Install automatically: Yes
```

## 📝 Pipeline Basics

### **Declarative Pipeline**

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
                sh 'npm install'
                sh 'npm run build'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Testing...'
                sh 'npm test'
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'Deploying...'
                sh './deploy.sh'
            }
        }
    }
}
```

### **Scripted Pipeline**

```groovy
// Jenkinsfile
node {
    stage('Build') {
        echo 'Building...'
        sh 'npm install'
        sh 'npm run build'
    }
    
    stage('Test') {
        echo 'Testing...'
        sh 'npm test'
    }
    
    stage('Deploy') {
        echo 'Deploying...'
        sh './deploy.sh'
    }
}
```

**Declarative vs Scripted:**

| Feature | Declarative | Scripted |
|---------|-------------|----------|
| **Syntax** | Structured, easier | Groovy, flexible |
| **Learning Curve** | Lower | Higher |
| **Use Case** | Standard pipelines | Complex logic |
| **Recommended** | ✅ For most cases | Advanced users |

## 🔨 Pipeline Components

### **Agent**

```groovy
pipeline {
    // Run on any available agent
    agent any
    
    // Or specific agent
    agent {
        label 'linux'
    }
    
    // Or Docker agent
    agent {
        docker {
            image 'node:18'
        }
    }
    
    // Or Kubernetes
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: maven
                    image: maven:3.8.6-openjdk-11
            '''
        }
    }
}
```

### **Environment Variables**

```groovy
pipeline {
    agent any
    
    environment {
        // Global environment variables
        APP_NAME = 'myapp'
        VERSION = '1.0.0'
        DOCKER_REGISTRY = 'myregistry.io'
        
        // Credentials
        DOCKER_CREDS = credentials('docker-hub-credentials')
        AWS_CREDS = credentials('aws-credentials')
    }
    
    stages {
        stage('Build') {
            environment {
                // Stage-specific variables
                BUILD_NUMBER = "${env.BUILD_NUMBER}"
            }
            steps {
                echo "Building ${APP_NAME} version ${VERSION}"
                echo "Build number: ${BUILD_NUMBER}"
            }
        }
    }
}
```

### **Parameters**

```groovy
pipeline {
    agent any
    
    parameters {
        // String parameter
        string(
            name: 'ENVIRONMENT',
            defaultValue: 'dev',
            description: 'Target environment'
        )
        
        // Choice parameter
        choice(
            name: 'DEPLOYMENT_TYPE',
            choices: ['blue-green', 'rolling', 'canary'],
            description: 'Deployment strategy'
        )
        
        // Boolean parameter
        booleanParam(
            name: 'RUN_TESTS',
            defaultValue: true,
            description: 'Run tests?'
        )
        
        // Password parameter
        password(
            name: 'API_KEY',
            description: 'API Key'
        )
    }
    
    stages {
        stage('Deploy') {
            steps {
                echo "Deploying to ${params.ENVIRONMENT}"
                echo "Strategy: ${params.DEPLOYMENT_TYPE}"
                
                script {
                    if (params.RUN_TESTS) {
                        echo "Running tests..."
                    }
                }
            }
        }
    }
}
```

### **Triggers**

```groovy
pipeline {
    agent any
    
    triggers {
        // Poll SCM every 5 minutes
        pollSCM('H/5 * * * *')
        
        // Cron schedule
        cron('H 2 * * *')  // Daily at 2 AM
        
        // GitHub webhook (configure in GitHub)
        githubPush()
        
        // Upstream job
        upstream(
            upstreamProjects: 'job1,job2',
            threshold: hudson.model.Result.SUCCESS
        )
    }
    
    stages {
        stage('Build') {
            steps {
                echo 'Building...'
            }
        }
    }
}
```

### **Post Actions**

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
    }
    
    post {
        always {
            // Always run
            echo 'Pipeline finished'
            cleanWs()
        }
        
        success {
            echo 'Build succeeded!'
            slackSend(
                color: 'good',
                message: "Build #${BUILD_NUMBER} succeeded"
            )
        }
        
        failure {
            echo 'Build failed!'
            emailext(
                subject: "Build Failed: ${JOB_NAME}",
                body: "Build #${BUILD_NUMBER} failed",
                to: 'team@example.com'
            )
        }
        
        unstable {
            echo 'Build unstable'
        }
        
        changed {
            echo 'Build status changed'
        }
    }
}
```

## 🚀 Real-World Pipelines

### **Node.js Application**

```groovy
pipeline {
    agent {
        docker {
            image 'node:18'
        }
    }
    
    environment {
        CI = 'true'
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test -- --coverage'
            }
            post {
                always {
                    junit 'test-results/**/*.xml'
                    publishHTML([
                        reportDir: 'coverage',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
        
        stage('Docker Build & Push') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def image = docker.build("myapp:${BUILD_NUMBER}")
                    docker.withRegistry('https://registry.example.com', 'docker-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    kubectl set image deployment/myapp \
                        myapp=myapp:${BUILD_NUMBER} \
                        -n production
                    kubectl rollout status deployment/myapp -n production
                '''
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(
                color: 'good',
                message: "✅ Build #${BUILD_NUMBER} succeeded: ${BUILD_URL}"
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: "❌ Build #${BUILD_NUMBER} failed: ${BUILD_URL}"
            )
        }
    }
}
```

### **Java Maven Application**

```groovy
pipeline {
    agent any
    
    tools {
        maven 'Maven3'
        jdk 'JDK11'
    }
    
    environment {
        MAVEN_OPTS = '-Xmx1024m'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/myorg/myapp.git',
                    credentialsId: 'github-credentials'
            }
        }
        
        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }
        
        stage('Publish Artifact') {
            steps {
                archiveArtifacts artifacts: '**/target/*.jar'
                
                // Nexus upload
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: 'nexus.example.com',
                    groupId: 'com.mycompany',
                    version: "${BUILD_NUMBER}",
                    repository: 'maven-releases',
                    credentialsId: 'nexus-credentials',
                    artifacts: [
                        [
                            artifactId: 'myapp',
                            file: 'target/myapp.jar',
                            type: 'jar'
                        ]
                    ]
                )
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                sh '''
                    scp target/myapp.jar user@staging:/opt/myapp/
                    ssh user@staging "sudo systemctl restart myapp"
                '''
            }
        }
        
        stage('Integration Tests') {
            steps {
                sh 'mvn verify -Pintegration-tests'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Deploy"
            }
            steps {
                sh '''
                    scp target/myapp.jar user@prod:/opt/myapp/
                    ssh user@prod "sudo systemctl restart myapp"
                '''
            }
        }
    }
}
```

### **Multi-Branch Pipeline**

```groovy
pipeline {
    agent any
    
    stages {
        stage('Build') {
            steps {
                echo "Building branch: ${env.BRANCH_NAME}"
                sh 'npm run build'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
            }
        }
        
        stage('Deploy to Dev') {
            when {
                not {
                    branch 'main'
                }
            }
            steps {
                echo "Deploying ${BRANCH_NAME} to dev"
                sh './deploy-dev.sh'
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                echo "Deploying to staging"
                sh './deploy-staging.sh'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
            }
            steps {
                echo "Deploying to production"
                sh './deploy-prod.sh'
            }
        }
    }
}
```

### **Docker Build & Deploy**

```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'myapp'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }
        
        stage('Test Image') {
            steps {
                script {
                    dockerImage.inside {
                        sh 'npm test'
                    }
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                sh """
                    # Trivy scan
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image \
                        ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }
        
        stage('Push Image') {
            steps {
                script {
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-credentials') {
                        dockerImage.push()
                        dockerImage.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh """
                        kubectl set image deployment/myapp \
                            myapp=${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                            -n production
                        
                        kubectl rollout status deployment/myapp -n production --timeout=5m
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh """
                    kubectl get pods -n production -l app=myapp
                    kubectl get svc -n production myapp
                """
            }
        }
    }
    
    post {
        always {
            sh "docker rmi ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} || true"
        }
    }
}
```

## 🔐 Credentials Management

### **Add Credentials**

```
Manage Jenkins → Credentials → System → Global credentials → Add Credentials
```

**Types:**
- Username with password
- Secret text
- Secret file
- SSH Username with private key
- Certificate

### **Use Credentials in Pipeline**

```groovy
pipeline {
    agent any
    
    stages {
        stage('Deploy') {
            steps {
                // Username & Password
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-hub',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )
                ]) {
                    sh 'docker login -u $USER -p $PASS'
                }
                
                // Secret Text
                withCredentials([
                    string(
                        credentialsId: 'api-key',
                        variable: 'API_KEY'
                    )
                ]) {
                    sh 'curl -H "Authorization: Bearer $API_KEY" ...'
                }
                
                // SSH Key
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'ssh-key',
                        keyFileVariable: 'SSH_KEY'
                    )
                ]) {
                    sh 'scp -i $SSH_KEY file user@server:/path'
                }
                
                // File
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh 'kubectl --kubeconfig=$KUBECONFIG get pods'
                }
            }
        }
    }
}
```

## 📊 Parallel Execution

```groovy
pipeline {
    agent any
    
    stages {
        stage('Parallel Tests') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
                    }
                }
                
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
                
                stage('E2E Tests') {
                    steps {
                        sh 'npm run test:e2e'
                    }
                }
                
                stage('Security Scan') {
                    steps {
                        sh 'npm audit'
                    }
                }
            }
        }
        
        stage('Deploy') {
            steps {
                echo 'All tests passed, deploying...'
            }
        }
    }
}
```

## 🎓 Best Practices

### **1. Version Control Jenkinsfile**

```bash
# Store Jenkinsfile in repository root
myproject/
├── Jenkinsfile
├── src/
└── tests/
```

### **2. Use Shared Libraries**

```groovy
// vars/buildAndTest.groovy
def call(String nodeVersion = '18') {
    pipeline {
        agent {
            docker {
                image "node:${nodeVersion}"
            }
        }
        stages {
            stage('Build') {
                steps {
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
            stage('Test') {
                steps {
                    sh 'npm test'
                }
            }
        }
    }
}

// Jenkinsfile
@Library('my-shared-library') _
buildAndTest('18')
```

### **3. Clean Workspace**

```groovy
post {
    always {
        cleanWs()
    }
}
```

### **4. Use Timeouts**

```groovy
options {
    timeout(time: 1, unit: 'HOURS')
}

stage('Deploy') {
    options {
        timeout(time: 10, unit: 'MINUTES')
    }
    steps {
        sh './deploy.sh'
    }
}
```

### **5. Retry Failed Steps**

```groovy
stage('Deploy') {
    steps {
        retry(3) {
            sh './deploy.sh'
        }
    }
}
```

## 🚨 Troubleshooting

### **Common Issues**

```groovy
// Permission denied on Docker socket
// Solution: Add jenkins user to docker group
sudo usermod -aG docker jenkins

// Pipeline syntax error
// Use: Pipeline Syntax Generator
// Manage Jenkins → Pipeline Syntax

// Workspace issues
// Clean workspace:
cleanWs()

// Agent not available
// Check: Manage Jenkins → Manage Nodes

// Plugin conflicts
// Update all plugins
// Restart Jenkins
```

## ✅ Quick Reference

```groovy
// Basic pipeline structure
pipeline {
    agent any
    environment { }
    parameters { }
    options { }
    triggers { }
    stages {
        stage('Name') {
            steps { }
        }
    }
    post { }
}

// Common steps
sh 'command'
echo 'message'
checkout scm
withCredentials([]) { }
docker.build()
junit '**/test-results/*.xml'
archiveArtifacts artifacts: '**/*.jar'
```

---

**Tiếp theo**: [GitLab CI/CD →](../gitlab/gitlab-ci.md)
