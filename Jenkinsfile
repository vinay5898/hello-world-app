pipeline {
    agent { node { label 'agent1' } }
    environment {
        AWS_ACCOUNT_ID = credentials('aws-account-id')
        AWS_DEFAULT_REGION = "ap-south-1" 
        IMAGE_REPO_NAME = "nodejsapp"
        IMAGE_TAG = "latest"
        REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"
        NOTIFY_EVENT_TOKEN = credentials('notify-token')
    }
   
    stages {
        stage('Logging into AWS ECR') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                }
            }
        }
        
        stage('Cloning Git') {
            steps {
                checkout scm     
            }
        }

        stage('Building image') {
            steps{
                script {
                    dockerImage = docker.build "${IMAGE_REPO_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Built approve notification') {
            options { timeout(time: 2, unit: 'MINUTES') }
            steps{
                script {
                    notifyEvents message: "Please <a href='${BUILD_URL}input'>click here</a> to go to console output of buildID ${BUILD_ID} to approve or Reject.", token: "$NOTIFY_EVENT_TOKEN"
                    //userInput = input submitter: '', message: "Do you approve the Build-${BUILD_TAG} ?"
                    env.node_name = input id: 'Agent', message: "Do you approve the Build-${BUILD_TAG} ?", submitter: 'admin', parameters: [choice(choices: ['agent1', 'agent2'], description: 'Which production machine?', name: 'agentSelect')]
                }
            }
        }

        stage('Pushing to ECR') {
            steps{  
                script {
                    sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:version${env.BUILD_NUMBER}"
                    sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:version${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Skype Notification') {
            steps{
                script {
                    //sh "echo Notification Sent"
                    notifyEvents message: '$BUILD_TAG | Built successfully', token: "$NOTIFY_EVENT_TOKEN"
                }
            }
        }

        stage('Deploy to Machine') {
            agent { label "${env.node_name}" }
            steps {
                script {
                    sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"
                    sh "docker pull ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:version${env.BUILD_NUMBER}"
                    sh 'docker ps -q --filter "name=nodejsapp" | grep -q . && docker kill nodejsapp || echo No running nodejsapp containers'
                    sh "sleep 10"
                    sh "docker run --rm --name nodejsapp -d -p 1111:8080 ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:version${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Container Notification') {
            steps{
                script {
                    notifyEvents message: 'Conrainer is up and running', token: "$NOTIFY_EVENT_TOKEN"
                }
            }
        }
    }
}
