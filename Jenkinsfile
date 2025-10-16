pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user/api-gateway"
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
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'mvn clean package -DskipTests'
                    } else {
                        bat 'mvn clean package -DskipTests'
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                echo "===== Deploying ${SERVICE_NAME} to EC2 ====="

                // Step 1: Create deploy directory and stop old process
                sshCommand remote: [name: 'ec2-server'], command: """
                    mkdir -p ${DEPLOY_DIR}
                    PID=\$(pgrep -f ${SERVICE_NAME}.jar) || true
                    if [ ! -z "\$PID" ]; then
                        echo "Stopping old ${SERVICE_NAME} (PID: \$PID)"
                        kill -9 \$PID
                    fi
                """

                // Step 2: Upload JAR
                sshPut remote: [name: 'ec2-server'], from: "target/${SERVICE_NAME}.jar", into: "${DEPLOY_DIR}/"

                // Step 3: Run application
                sshCommand remote: [name: 'ec2-server'], command: """
                    nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar --server.port=${SERVER_PORT} > ${DEPLOY_DIR}/${LOG_FILE} 2>&1 &
                    echo "${SERVICE_NAME} started successfully!"
                """
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
