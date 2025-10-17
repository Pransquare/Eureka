pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user/deployment"
        EC2_HOST = "13.60.33.154"
        SERVICE_NAME = "eureka-server"
        SERVER_PORT = "8761"
        LOG_FILE = "eureka-server.log"
        SSH_CREDENTIALS_ID = "ec2-key" // Jenkins SSH credentials
    }

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build') {
            steps {
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

        stage('Deploy to EC2') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'ec2-ssh-server',
                            transfers: [
                                sshTransfer(
                                    sourceFiles: "target/${SERVICE_NAME}.jar",
                                    removePrefix: 'target',       // Put JAR directly in DEPLOY_DIR
                                    remoteDirectory: DEPLOY_DIR
                                )
                            ],
                            usePromotionTimestamp: false,
                            verbose: true,
                            execCommand: """
                                echo 'Starting deployment on EC2...'
                                mkdir -p ${DEPLOY_DIR}
                                echo 'Killing existing process if any'
                                pkill -f ${SERVICE_NAME}.jar || true
                                echo 'Starting new process'
                                nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar --server.port=${SERVER_PORT} > ${DEPLOY_DIR}/${LOG_FILE} 2>&1 & || true
                                sleep 3
                                echo 'Deployment script completed'
                                echo 'Check logs: ${DEPLOY_DIR}/${LOG_FILE}'
                            """
                        )
                    ]
                )
            }
        }
    }

    post {
        success {
            echo " Deployment completed successfully! Check EC2 logs at ${DEPLOY_DIR}/${LOG_FILE}"
        }
        unstable {
            echo "Deployment completed but with warnings. Check Jenkins console and EC2 logs."
        }
        failure {
            echo " Deployment failed. Check Jenkins console and EC2 logs."
        }
    }
}
