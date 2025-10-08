pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
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

        stage('Deploy') {
            steps {
                bat 'taskkill /F /IM java.exe || exit 0'
                bat "if not exist %DEPLOY_DIR% mkdir %DEPLOY_DIR%"
                bat "xcopy target\\*.jar %DEPLOY_DIR% /Y /Q"
                bat "start cmd /c java -jar %DEPLOY_DIR%\\%SERVICE_NAME%.jar --server.port=%SERVICE_PORT%"
            }
        }
    }
}
