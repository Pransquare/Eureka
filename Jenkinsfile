pipeline {
    agent any

    environment {
        DEPLOY_DIR = "/home/ec2-user"
        EC2_HOST = "51.21.200.23"
        SERVICE_NAME = "eureka-server"
        PEM_PATH = "C:\\ProgramData\\Jenkins\\.ssh\\krishna.pem"
    }

    tools {
        jdk 'Java17'
        maven 'Maven3'
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
                withEnv(["PATH=${tool 'Git'}/bin:${env.PATH}"]) {
                    bat """
                    echo ===== Copying JAR to EC2 =====
                    scp -i "${PEM_PATH}" -o StrictHostKeyChecking=no target\\${SERVICE_NAME}.jar ec2-user@${EC2_HOST}:${DEPLOY_DIR}/

                    echo ===== Stopping old Eureka instance if running =====
                    ssh -i "${PEM_PATH}" -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} "pkill -f ${SERVICE_NAME}.jar || true"

                    echo ===== Starting new Eureka instance =====
                    ssh -i "${PEM_PATH}" -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} "nohup java -jar ${DEPLOY_DIR}/${SERVICE_NAME}.jar --server.port=8761 > ${DEPLOY_DIR}/eureka.log 2>&1 &"

                    echo ✅ Deployment completed successfully!
                    """
                }
            }
        }
    }

    post {
        failure {
            echo "❌ Deployment failed. Check Jenkins console logs for details."
        }
    }
}
