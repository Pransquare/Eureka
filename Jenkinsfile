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
        S3_BUCKET = "eureka-deployment-2025"
        REGION = "eu-north-1"
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

        stage('Upload JAR to S3') {
            steps {
                withAWS(credentials: 'aws-jenkins-creds', region: "${REGION}") {
                    s3Upload(bucket: "${S3_BUCKET}", path: "${SERVICE_NAME}.jar", file: "target\\${SERVICE_NAME}.jar")
                }
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
                        Write-Output '--- Starting deployment to EC2 ---'

                        \$pass = ConvertTo-SecureString '${EC2_PASSWORD}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential('Administrator', \$pass)
                        \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
                        \$session = New-PSSession -ComputerName 13.53.193.215 -UseSSL -Credential $cred -Authentication Basic -SessionOption $sessOption


                        Invoke-Command -Session \$session -ScriptBlock {
                            if (-not (Test-Path -Path '${DEPLOY_DIR}')) {
                                New-Item -Path '${DEPLOY_DIR}' -ItemType Directory | Out-Null
                            }
                        }

                        Invoke-Command -Session \$session -ScriptBlock {
                            Get-Process java -ErrorAction SilentlyContinue | Stop-Process -Force
                        }

                        Copy-Item -ToSession \$session -Path "C:\\Jenkins\\workspace\\Eureka-deploy-AWS\\target\\${SERVICE_NAME}.jar" -Destination "${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -Force

                        Invoke-Command -Session \$session -ScriptBlock {
                            Start-Process -FilePath 'java' -ArgumentList "-jar ${DEPLOY_DIR}\\${SERVICE_NAME}.jar --server.port=${SERVICE_PORT}" -NoNewWindow
                        }

                        Remove-PSSession \$session
                    """
                }
            }
        }
    }
}
