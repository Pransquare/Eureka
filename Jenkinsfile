pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user/services/eureka-server"
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
                    if (isUnix()) {
                        sh 'mvn clean package -DskipTests'
                    } else {
                        bat 'mvn clean package -DskipTests'
                    }
                }
            }
        }

    stage('Prepare Deploy Script') {
    steps {
        echo "===== Creating deploy.sh script ====="
        script {
            writeFile file: 'target/deploy.sh', text: """
#!/bin/bash
SERVICE_NAME=${SERVICE_NAME}
DEPLOY_DIR=/home/ec2-user/services/\$SERVICE_NAME
LOG_FILE=\${SERVICE_NAME}.log

mkdir -p \$DEPLOY_DIR
pkill -f \${SERVICE_NAME}.jar || true
mv \${SERVICE_NAME}.jar \$DEPLOY_DIR/\${SERVICE_NAME}.jar || true
nohup java -jar \$DEPLOY_DIR/\${SERVICE_NAME}.jar --server.port=${SERVER_PORT} > \$DEPLOY_DIR/\$LOG_FILE 2>&1 &
echo "Deployment completed successfully."
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
                    configName: 'ec2-server',
                    transfers: [
                        sshTransfer(
                            sourceFiles: "target/${SERVICE_NAME}.jar,target/deploy.sh",
                            removePrefix: "target",
                            remoteDirectory: "/home/ec2-user/services/${SERVICE_NAME}",
                            execCommand: """
chmod +x /home/ec2-user/services/${SERVICE_NAME}/deploy.sh
/home/ec2-user/services/${SERVICE_NAME}/deploy.sh
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
            echo "✅ Deployment completed successfully! ${SERVICE_NAME} is running on port ${SERVER_PORT}"
        }
        failure {
            echo "❌ Deployment failed. Check Jenkins console logs for details."
        }
    }
}
