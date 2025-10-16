pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user/services/eureka-server"
        SERVICE_NAME = "eureka-server"
        SERVER_PORT = "8761"
        LOG_FILE = "eureka-server.log"
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
                    if (isUnix()) {
                        sh "mvn clean package -DskipTests"
                    } else {
                        bat "mvn clean package -DskipTests"
                    }
                }
            }
        }

        stage('Prepare Deploy Script') {
            steps {
                script {
                    writeFile file: 'deploy.sh', text: """#!/bin/bash
mkdir -p ${DEPLOY_DIR}
PID=\$(pgrep -f ${SERVICE_NAME}.jar) || true
if [ ! -z "\$PID" ]; then
  kill -9 \$PID
fi
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
            configName: 'ec2-server',
            transfers: [
                sshTransfer(
                    sourceFiles: "target/${SERVICE_NAME}.jar, deploy.sh",
                    removePrefix: '',
                    remoteDirectory: "/home/ec2-user",
                    execCommand: """
                        mkdir -p /home/ec2-user/services/eureka-server
                        mv deploy.sh ${SERVICE_NAME}.jar services/eureka-server/
                        chmod +x services/eureka-server/deploy.sh
                        services/eureka-server/deploy.sh
                    """
                )
            ],
            usePromotionTimestamp: false,
            verbose: true
        )
    ]
)

            ]
        )
    }
}

    }

    post {
        success {
            echo "Deployment completed successfully! ${SERVICE_NAME} is running on port ${SERVER_PORT}"
        }
        failure {
            echo "Deployment failed. Check Jenkins console logs for details."
        }
    }
}
