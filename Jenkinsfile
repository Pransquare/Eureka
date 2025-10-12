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

        stage('Deploy to EC2 via WinRM HTTPS') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'ec2-admin-creds',
                    usernameVariable: 'EC2_USER',
                    passwordVariable: 'EC2_PASSWORD'
                )]) {
                    powershell """
                        Write-Host '--- Connecting to EC2 via WinRM HTTPS ---'

                        # Convert password and create credentials
                        \$secpasswd = ConvertTo-SecureString '${EC2_PASSWORD}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential ('${EC2_USER}', \$secpasswd)
                        \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck

                        # Create PSSession to EC2
                        \$session = New-PSSession -ComputerName ${EC2_HOST} -UseSSL -Credential \$cred -SessionOption \$sessOption

                        Write-Host '--- Creating deployment directory if not exists ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            if (-not (Test-Path '${DEPLOY_DIR}')) {
                                New-Item -ItemType Directory -Path '${DEPLOY_DIR}' | Out-Null
                            }
                        }

                        Write-Host '--- Stopping existing Java process ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            Get-Process java -ErrorAction SilentlyContinue | Stop-Process -Force
                        }

                        Write-Host '--- Copying JAR to remote EC2 via WinRM ---'
                        Copy-Item -Path "target\\${SERVICE_NAME}.jar" -Destination "${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -ToSession \$session -Force

                        Write-Host '--- Starting the Spring Boot service ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            Start-Process -FilePath "java" -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar --server.port=${SERVICE_PORT}" -NoNewWindow
                        }

                        Write-Host '--- Cleaning up session ---'
                        Remove-PSSession \$session
                    """
                }
            }
        }
    }
}
