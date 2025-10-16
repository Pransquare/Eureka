pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user/services/eureka-server"
        EC2_HOST = "13.53.39.170"
        SERVICE_NAME = "eureka-server"
        SERVER_PORT = "8761"
        LOG_FILE = "eureka-server.log"
        SSH_CREDENTIALS_ID = "ec2-ssh-key" // Jenkins SSH credentials ID
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

        stage('Prepare Deploy Script') {
            steps {
                script {
                    writeFile file: 'deploy.sh', text: """
                        #!/bin/bash
                        mkdir -p ${DEPLOY_DIR}
                        pkill -f ${SERVICE_NAME}.jar || true
                        nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar --server.port=${SERVER_PORT} > ${DEPLOY_DIR}/${LOG_FILE} 2>&1 &
                        echo "Deployment completed"
                    """
                }
            }
        }

       stage('Deploy to EC2') {
    steps {
        sshPublisher(
            publishers: [
                sshPublisherDesc(
                    configName: 'ec2-server', // Jenkins SSH configuration
                    transfers: [
                        sshTransfer(
                            sourceFiles: "target/${SERVICE_NAME}.jar, deploy.sh",
                            removePrefix: '',
                            remoteDirectory: "/home/ec2-user/tmp", // upload both here
                            execCommand: """
                                mkdir -p ${DEPLOY_DIR}
                                mv /home/ec2-user/tmp/deploy.sh /home/ec2-user/tmp/${SERVICE_NAME}.jar ${DEPLOY_DIR}/
                                chmod +x ${DEPLOY_DIR}/deploy.sh
                                ${DEPLOY_DIR}/deploy.sh
                            """
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
            echo " Deployment completed successfully! ${SERVICE_NAME} is running on port ${SERVER_PORT}"
        }
        failure {
            echo " Deployment failed. Check Jenkins console logs for details."
        }
    }
}
