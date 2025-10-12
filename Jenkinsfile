pipeline {
    agent any

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        EC2_HOST = "13.53.193.215"
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
                // Use Jenkins credentials securely
                withCredentials([usernamePassword(
                    credentialsId: 'ec2-admin-creds',
                    usernameVariable: 'EC2_USER',
                    passwordVariable: 'EC2_PASSWORD'
                )]) {
                    powershell """
                        # Convert password to secure string
                        \$secpasswd = ConvertTo-SecureString '${EC2_PASSWORD}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential ('${EC2_USER}', \$secpasswd)

                        # Session options to skip certificate checks
                        \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck

                        # Create a WinRM session
                        \$session = New-PSSession -ComputerName ${EC2_HOST} -UseSSL -Credential \$cred -SessionOption \$sessOption

                        # 1️⃣ Create deployment directory if it doesn't exist
                        Invoke-Command -Session \$session -ScriptBlock {
                            if (-not (Test-Path -Path '${DEPLOY_DIR}')) {
                                New-Item -Path '${DEPLOY_DIR}' -ItemType Directory | Out-Null
                            }
                        }

                        # 2️⃣ Stop any running Java processes
                        Invoke-Command -Session \$session -ScriptBlock {
                            Get-Process java -ErrorAction SilentlyContinue | Stop-Process -Force
                        }

                        # 3️⃣ Copy the JAR file to EC2
                        Copy-Item -Path 'target\\${SERVICE_NAME}.jar' -Destination "${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -ToSession \$session -Force

                        # 4️⃣ Start the Spring Boot application
                        Invoke-Command -Session \$session -ScriptBlock {
                            Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar --server.port=${SERVICE_PORT}" -NoNewWindow
                        }

                        # Close the session
                        Remove-PSSession \$session
                    """
                }
            }
        }
    }
}
