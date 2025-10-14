pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        DEPLOY_DIR = "C:\\ProgramData\\Jenkins\\.jenkins\\workspace\\Deploy-Eureka"
        PEM_KEY = "C:\\ProgramData\\Jenkins\\.ssh\\krishna.pem"
        EC2_USER = "ec2-user"
        EC2_HOST = "13.60.47.188"
        REMOTE_DIR = "/home/ec2-user"
        JAR_NAME = "eureka-server.jar"
        SERVER_PORT = "8761"
        LOG_FILE = "eureka.log"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build') {
            steps {
                bat "mvn clean package -DskipTests"
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    echo "===== Testing EC2 SSH Connection ====="
                    def sshTest = bat(
                        script: """ssh -i "${PEM_KEY}" -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "echo Connected" """,
                        returnStatus: true
                    )
                    if (sshTest != 0) {
                        error("❌ EC2 is not reachable — please check network or key permissions")
                    }

                    echo "===== Copying JAR to EC2 (with retry) ====="
                    retry(3) {
                        bat """scp -i "${PEM_KEY}" -o StrictHostKeyChecking=no ${DEPLOY_DIR}\\target\\${JAR_NAME} ${EC2_USER}@${EC2_HOST}:${REMOTE_DIR}/"""
                    }

                    echo "===== Stopping Old Eureka Instance ====="
                    bat """ssh -i "${PEM_KEY}" -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "pkill -f ${JAR_NAME} || true" """

                    echo "===== Starting New Eureka Instance ====="
                    retry(2) {
                        bat """ssh -i "${PEM_KEY}" -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "nohup java -jar ${REMOTE_DIR}/${JAR_NAME} --server.port=${SERVER_PORT} > ${REMOTE_DIR}/${LOG_FILE} 2>&1 &" """
                    }

                    echo "===== Waiting for Eureka to Start ====="
                    sleep 15  // give it a few seconds to boot up

                    echo "===== Checking if Eureka is Running ====="
                    def checkApp = bat(
                        script: """ssh -i "${PEM_KEY}" -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "netstat -tulnp | grep ${SERVER_PORT}" """,
                        returnStatus: true
                    )
                    if (checkApp != 0) {
                        error("❌ Eureka is not running on port ${SERVER_PORT}. Check logs below.")
                    } else {
                        echo "✅ Eureka is running successfully on port ${SERVER_PORT}!"
                    }

                    echo "===== Showing Latest 20 Lines from Eureka Logs ====="
                    bat """ssh -i "${PEM_KEY}" -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "tail -n 20 ${REMOTE_DIR}/${LOG_FILE}" """
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Deployment successful! Access Eureka at: http://${EC2_HOST}:${SERVER_PORT}"
        }
        failure {
            echo "❌ Deployment failed. Check EC2 connection, JAR integrity, or logs."
        }
    }
}
