pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-2'
        ECR_REGISTRY = '386318011177.dkr.ecr.us-east-2.amazonaws.com'
        ECR_IMAGE = '386318011177.dkr.ecr.us-east-2.amazonaws.com/test'
        EC2_IP = '3.15.43.184'
        EC2_USER = 'ubuntu'
        DockerComposeFile = 'docker-compose.yml'
        DotEnvFile = '.env'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                bat 'docker build -t app:%BUILD_NUMBER% .'
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    bat '''
                    aws ecr get-login-password --region %AWS_REGION% > ecr_pass.txt
                    type ecr_pass.txt | docker login --username AWS --password-stdin %ECR_REGISTRY%
                    del ecr_pass.txt
                    '''
                }
            }
        }

        stage('Tag and Push Image') {
            steps {
                bat '''
                docker tag app:%BUILD_NUMBER% %ECR_IMAGE%:%BUILD_NUMBER%
                docker tag app:%BUILD_NUMBER% %ECR_IMAGE%:latest
                docker push %ECR_IMAGE%:%BUILD_NUMBER%
                docker push %ECR_IMAGE%:latest
                '''
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    sshUserPrivateKey(credentialsId: 'ec2', keyFileVariable: 'EC2_KEY', usernameVariable: 'EC2_USER'),
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']
                ]) {
                    bat '''
                    icacls "%EC2_KEY%" /inheritance:r
                    icacls "%EC2_KEY%" /grant:r "%USERNAME%:F"
                    aws ecr get-login-password --region %AWS_REGION% > ecr_pass.txt
                    type ecr_pass.txt | ssh -i "%EC2_KEY%" -o StrictHostKeyChecking=no %EC2_USER%@%EC2_IP% "docker login --username AWS --password-stdin %ECR_REGISTRY%"
                    del ecr_pass.txt

                    scp -i "%EC2_KEY%" -o StrictHostKeyChecking=no "%DotEnvFile%" "%DockerComposeFile%" %EC2_USER%@%EC2_IP%:/home/ubuntu/

                    ssh -i "%EC2_KEY%" -o StrictHostKeyChecking=no %EC2_USER%@%EC2_IP% "docker compose -f /home/ubuntu/%DockerComposeFile% --env-file /home/ubuntu/%DotEnvFile% down || true"
                    ssh -i "%EC2_KEY%" -o StrictHostKeyChecking=no %EC2_USER%@%EC2_IP% "docker compose -f /home/ubuntu/%DockerComposeFile% --env-file /home/ubuntu/%DotEnvFile% pull"
                    ssh -i "%EC2_KEY%" -o StrictHostKeyChecking=no %EC2_USER%@%EC2_IP% "docker compose -f /home/ubuntu/%DockerComposeFile% --env-file /home/ubuntu/%DotEnvFile% up -d"
                    '''
                }
            }
        }
    }
}