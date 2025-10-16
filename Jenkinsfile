pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user"
        EC2_HOST = "13.53.39.170"
        SERVICE_NAME = "eureka-server"
        SERVER_PORT = "8761"
        LOG_FILE = "eureka-server.log"
        SSH_CREDENTIALS_ID = "ec2-ssh-key" // Jenkins SSH credentials
    }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "===== Cloning repository from GitHub ====="
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build') {
            steps {
                echo "===== Building project using Maven ====="
                script {
                    def mvnCmd = 'mvn clean package -DskipTests'
                    if (isUnix()) {
                        sh mvnCmd
                    } else {
                        bat mvnCmd
                    }
                }
            }
        }

        stage('Prepare Deploy Script') {
            steps {
                echo "===== Creating deploy.sh script ====="
                script {
                    writeFile file: 'deploy.sh', text: """
                        #!/bin/bash
                        echo "===== Starting Deployment ====="
                        mkdir -p ${DEPLOY_DIR}
                        
                        echo "Stopping existing instance (if running)..."
                        pkill -f ${SERVICE_NAME}.jar || true
                        
                        echo "Starting new instance..."
                        nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar --server.port=${SERVER_PORT} > ${DEPLOY_DIR}/${LOG_FILE} 2>&1 &
                        
                        echo "===== Deployment completed successfully! ====="
                        ps -ef | grep ${SERVICE_NAME}.jar | grep -v grep
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "===== Deploying application to EC2 ====="
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'ec2-server', // Must match name under Manage Jenkins → Publish over SSH
                            transfers: [
                                sshTransfer(
                                    // ✅ Use double quotes so variables expand correctly
                                    sourceFiles: "target/${SERVICE_NAME}.jar, deploy.sh",
                                    removePrefix: 'target',
                                    remoteDirectory: "${DEPLOY_DIR}",
                                    execCommand: "chmod +x ${DEPLOY_DIR}/deploy.sh && ${DEPLOY_DIR}/deploy.sh"
                                )
                            ],
                            usePromotionTimestamp: false,
                            verbose: true
                        )
                    ]
                )
            }
        }
    }

    post {
        success {
            echo "✅ Deployment completed successfully! ${SERVICE_NAME} is running on port ${SERVER_PORT}"
        }
        failure {
            echo "❌ Deployment failed. Check Jenkins console logs for details."
        }
    }
}
