pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        SERVICE_NAME = "nems-eureka-server"
        SERVICE_PORT = "8761"
        DEPLOY_DIR = "/opt/nems"
        EC2_HOST = "13.60.47.188"
        SSH_CREDENTIAL_ID = "ec2-linux-key"  // Jenkins SSH credential ID
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out Eureka repository..."
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build & Package') {
            steps {
                echo "Building Eureka service..."
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent([env.SSH_CREDENTIAL_ID]) {
                    sh """
                    # Create deployment and logs folder on EC2
                    ssh -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} "mkdir -p ${DEPLOY_DIR}/logs"

                    # Stop old Eureka service if running
                    ssh ec2-user@${EC2_HOST} "pkill -f ${SERVICE_NAME}.jar || true"

                    # Copy new JAR to EC2
                    scp -o StrictHostKeyChecking=no target/${SERVICE_NAME}.jar ec2-user@${EC2_HOST}:${DEPLOY_DIR}/

                    # Start Eureka service in background
                    ssh ec2-user@${EC2_HOST} "nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar --server.port=${SERVICE_PORT} > ${DEPLOY_DIR}/logs/${SERVICE_NAME}.log 2>&1 &"
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Eureka deployment pipeline finished"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
