pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        EC2_HOST = "13.53.193.215"
        EC2_USER = "Administrator"       // Your EC2 Windows user
        EC2_PASSWORD = "d8%55Ir.%Z!hNR%VgUe-07OYX0ujLy;S" // Or use Jenkins credentials for security
        DEPLOY_DIR = "C:\\Apps\\eureka-server"
        SERVICE_NAME = "eureka-server"
        SERVICE_PORT = "8761"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Pransquare/Eureka.git'
            }
        }

        stage('Build') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    // 1️⃣ Copy the built JAR to EC2 (using WinRM)
                    bat """
                    psexec \\\\${EC2_HOST} -u ${EC2_USER} -p ${EC2_PASSWORD} cmd /c "if not exist ${DEPLOY_DIR} mkdir ${DEPLOY_DIR}"
                    """

                    bat """
                    copy target\\*.jar \\\\${EC2_HOST}\\${DEPLOY_DIR}\\
                    """

                    // 2️⃣ Stop existing service (if running)
                    bat """
                    psexec \\\\${EC2_HOST} -u ${EC2_USER} -p ${EC2_PASSWORD} taskkill /F /IM java.exe || exit 0
                    """

                    // 3️⃣ Start new instance
                    bat """
                    psexec \\\\${EC2_HOST} -u ${EC2_USER} -p ${EC2_PASSWORD} cmd /c "start java -jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar --server.port=${SERVICE_PORT}"
                    """
                }
            }
        }
    }
}
