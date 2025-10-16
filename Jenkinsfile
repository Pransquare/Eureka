pipeline {
    agent any

    environment {
        SERVICE_NAME = "eureka-server"
        SERVER_PORT = "8761"
    }

    stages {
        stage('Checkout SCM') {
            steps {
                echo "===== Cloning repository from GitHub ====="
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build') {
            steps {
                echo "===== Building project using Maven ====="
                bat "mvn clean package -DskipTests"
            }
        }

        stage('Prepare Deploy Script') {
            steps {
                echo "===== Creating deploy.sh script ====="
                script {
                    writeFile file: "target/deploy.sh", text: """
#!/bin/bash
SERVICE_NAME=${SERVICE_NAME}
DEPLOY_DIR=/home/ec2-user/services/\$SERVICE_NAME
LOG_FILE=\${SERVICE_NAME}.log

# Create directories if not exist
mkdir -p \$DEPLOY_DIR

# Stop running service if exists
pkill -f \${SERVICE_NAME}.jar || true

# Move JAR to deploy directory
mv \${SERVICE_NAME}.jar \$DEPLOY_DIR/\${SERVICE_NAME}.jar || true

# Start service
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
                            configName: 'ec2-server',  // Your Jenkins SSH config
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
            echo "Eureka service deployed successfully!"
        }
        unstable {
            echo "Deployment completed with warnings."
        }
        failure {
            echo "Deployment failed."
        }
    }
}
