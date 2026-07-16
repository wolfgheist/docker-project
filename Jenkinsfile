pipeline {
    agent any

    environment {
        AWS_REGION       = 'us-east-2'
        AWS_ACCOUNT_ID   = '386318011177'
        ECR_REPOSITORY   = 'test'
        ECR_URI          = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
        EC2_IP           = '3.15.43.184'
        DockerComposeFile = 'docker-compose.yml'
        DotEnvFile        = '.env'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t app:${BUILD_NUMBER} .'
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh '''
                        aws ecr get-login-password --region "$AWS_REGION" | \
                        docker login --username AWS --password-stdin "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
                    '''
                }
            }
        }

        stage('Tag and Push Image') {
            steps {
                sh '''
                    docker tag app:${BUILD_NUMBER} "$ECR_URI:${BUILD_NUMBER}"
                    docker tag app:${BUILD_NUMBER} "$ECR_URI:latest"
                    docker push "$ECR_URI:${BUILD_NUMBER}"
                    docker push "$ECR_URI:latest"
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'ec2', keyFileVariable: 'EC2_KEY', usernameVariable: 'EC2_USER'),
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']
                ]) {
                    sh '''
                        scp -i "$EC2_KEY" -o StrictHostKeyChecking=no "$DotEnvFile" "$DockerComposeFile" "$EC2_USER@$EC2_IP:/home/ubuntu/"

                        PASSWORD=$(aws ecr get-login-password --region "$AWS_REGION")
                        printf '%s' "$PASSWORD" | ssh -i "$EC2_KEY" -o StrictHostKeyChecking=no "$EC2_USER@$EC2_IP" \
                            "docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

                        ssh -i "$EC2_KEY" -o StrictHostKeyChecking=no "$EC2_USER@$EC2_IP" \
                            "docker compose -f /home/ubuntu/$DockerComposeFile --env-file /home/ubuntu/$DotEnvFile down || true"

                        ssh -i "$EC2_KEY" -o StrictHostKeyChecking=no "$EC2_USER@$EC2_IP" \
                            "docker compose -f /home/ubuntu/$DockerComposeFile --env-file /home/ubuntu/$DotEnvFile pull"

                        ssh -i "$EC2_KEY" -o StrictHostKeyChecking=no "$EC2_USER@$EC2_IP" \
                            "docker compose -f /home/ubuntu/$DockerComposeFile --env-file /home/ubuntu/$DotEnvFile up -d"
                    '''
                }
            }
        }
    }
}