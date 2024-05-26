Install  Java
Download Jenkins.war file and run it on terminal with java -jar Jenkins.war
Create Jobs
Run the Job with Build Now button


## Common pipeline script
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY_CREDENTIALS = 'docker-registry-credentials'
        DOCKER_IMAGE_NAME = 'your-docker-image-name'
        DOCKER_IMAGE_TAG = 'latest'
        ECR_REPOSITORY_URI = '123456789012.dkr.ecr.us-west-2.amazonaws.com/your-repository'
        ECS_CLUSTER_NAME = 'your-ecs-cluster-name'
        ECS_TASK_DEFINITION = 'your-task-definition-name'
        ECS_SERVICE_NAME = 'your-service-name'
    }

    stages {
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}")
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: DOCKER_REGISTRY_CREDENTIALS, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh "aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin ${ECR_REPOSITORY_URI}"
                        docker.withRegistry("${ECR_REPOSITORY_URI}", 'ecr') {
                            docker.image("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}").push()
                        }
                    }
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                script {
                    // Update ECS task definition with new image
                    sh "aws ecs register-task-definition --family ${ECS_TASK_DEFINITION} --container-definitions '[{\"name\":\"your-container-name\",\"image\":\"${ECR_REPOSITORY_URI}:${DOCKER_IMAGE_TAG}\",\"essential\":true,\"portMappings\":[{\"containerPort\":8080}]}]'"
                    
                    // Update ECS service with new task definition
                    sh "aws ecs update-service --cluster ${ECS_CLUSTER_NAME} --service ${ECS_SERVICE_NAME} --task-definition ${ECS_TASK_DEFINITION}"
                }
            }
        }
    }
}



## deployment.yaml for kubernates
apiVersion: apps/v1
kind: Deployment
metadata:
  name: your-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: your-app
  template:
    metadata:
      labels:
        app: your-app
    spec:
      containers:
      - name: your-container
        image: 123456789012.dkr.ecr.us-west-2.amazonaws.com/your-repository:latest
        ports:
        - containerPort: 8080



## EKS deployment
pipeline {
    agent any

    environment {
        KUBECONFIG = '/path/to/kubeconfig'
        KUBE_NAMESPACE = 'your-namespace'
    }

    stages {
        stage('Deploy to EKS') {
            steps {
                script {
                    // Apply Kubernetes manifests to EKS
                    sh "kubectl apply -f deployment.yaml --namespace=${KUBE_NAMESPACE} --kubeconfig=${KUBECONFIG}"
                }
            }
        }
    }
}

