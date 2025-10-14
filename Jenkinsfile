pipeline {
    agent any
    environment {
        DEPLOY_DIR = "/opt/nems"
        SERVICE_NAME = "eureka"
        EC2_HOST = "13.60.47.188"
    }
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/Pransquare/Eureka.git', branch: 'master'
            }
        }
        stage('Build') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }
        stage('Deploy to EC2') {
            steps {
                sshagent(['EC2_SSH_CREDENTIAL_ID']) {
                    bat """
                    ssh -o StrictHostKeyChecking=no ec2-user@%EC2_HOST% "mkdir -p %DEPLOY_DIR%"
                    scp target/%SERVICE_NAME%.jar ec2-user@%EC2_HOST%:%DEPLOY_DIR%/
                    ssh ec2-user@%EC2_HOST% "pkill -f %SERVICE_NAME%.jar || true"
                    ssh ec2-user@%EC2_HOST% "nohup java -jar %DEPLOY_DIR%/%SERVICE_NAME%.jar --server.port=8761 > %DEPLOY_DIR%/logs/%SERVICE_NAME%.log 2>&1 &"
                    """
                }
            }
        }
    }
}
