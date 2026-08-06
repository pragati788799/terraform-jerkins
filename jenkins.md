## Installation 
## Docker 
```
sudo apt install docker.io -y
sudo systemctl start docker
sudo usermod -aG docker jenkins
sudo usermod -aG docker ubuntu
newgrp docker
```
## Terraform installation 
- https://developer.hashicorp.com/terraform/install

## AWS Cli Installation 
```
sudo snap install aws-cli --classic
chmod 777 /var/run/docker.sock
```
# attach iam role if permission denied error occurred 

## jenkins 3 tire application with the help of jenkins pipeline 
```groovy 
pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "sneha788799"
        REPO_NAME      = "terraform-jerkins"
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/pragati788799/terraform-jerkins.git'
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh "docker build -t ${DOCKERHUB_USER}/${REPO_NAME}:frontend ./frontend"
            }
        }

        stage('Build Backend Image') {
            steps {
                sh "docker build --no-cache -t ${DOCKERHUB_USER}/${REPO_NAME}:backend ./backend"
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-cred',
                        usernameVariable: 'USERNAME',
                        passwordVariable: 'PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$PASSWORD" | docker login -u "$USERNAME" --password-stdin
                    '''
                }
            }
        }

        stage('Push Frontend Image') {
            steps {
                sh "docker push ${DOCKERHUB_USER}/${REPO_NAME}:frontend"
            }
        }

        stage('Push Backend Image') {
            steps {
                sh "docker push ${DOCKERHUB_USER}/${REPO_NAME}:backend"
            }
        }
    }

    post {
        always {
            sh "docker logout || true"
        }

        success {
            echo "Docker images built and pushed successfully."
        }

        failure {
            echo "Pipeline failed."
        }
    }
}
````

```groovy
pipeline {
    agent any

    tools {
        terraform 'terraform'
    }

    parameters {
        choice(
            name: 'action',
            choices: ['apply', 'destroy'],
            description: 'Select Terraform Action'
        )
    }

    stages {

        stage('Code Pull') {
            steps {
                git branch: 'main',
                url: 'https://github.com/pragati788799/terraform-jerkins.git'
            }
        }

        stage('Terraform Init') {
            steps {
                dir('EKS-TF') {
                    withCredentials([
                        aws(
                            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                            credentialsId: 'aws-cred',
                            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                        )
                    ]) {
                        sh 'terraform init'
                    }
                }
            }
        }

        stage('Terraform Validate') {
            steps {
                dir('EKS-TF') {
                    sh 'terraform validate'
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                dir('EKS-TF') {
                    withCredentials([
                        aws(
                            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                            credentialsId: 'aws-cred',
                            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                        )
                    ]) {
                        sh 'terraform plan'
                    }
                }
            }
        }

        stage('Terraform Action') {
            steps {
                dir('EKS-TF') {
                    withCredentials([
                        aws(
                            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                            credentialsId: 'aws-cred',
                            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                        )
                    ]) {
                        sh "terraform ${params.action} -auto-approve"
                    }
                }
            }
        }

        stage('Trigger Job3') {
            when {
                expression { params.action == 'apply' }
            }
            steps {
                build job: 'job3', wait: true
            }
        }

        stage('Completed') {
            steps {
                echo 'Back from Job2'
            }
        }
    }
}


```

```groovy
pipeline {
    agent { label 'eksagent' }

    environment {
        AWS_DEFAULT_REGION = 'ap-south-1'
    }

    stages {

        stage('Code Pull') {
            steps {
                git branch: 'main',
                url: 'https://github.com/pragati788799/terraform-jerkins.git'
            }
        }

        stage('Deploy To EKS') {
            steps {

                withCredentials([
                    aws(
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        credentialsId: 'aws-cred',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {

                    sh '''
                    aws eks update-kubeconfig \
                    --region ap-south-1 \
                    --name EKS_CLOUD

                    kubectl get nodes

                    cd backend
                    kubectl apply -f backend.yaml
                    cd ../frontend 
                    kubectl apply -f frontend.yaml
                    cd ..

                    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

                    kubectl apply -f ingress.yaml

                    kubectl get pods

                    kubectl get svc
                    '''
                }
            }
        }
    }
}
```
        
