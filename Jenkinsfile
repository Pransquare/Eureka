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
        S3_BUCKET = "eureka-deployment-2025" // Replace with your S3 bucket
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

        stage('Upload to S3') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                  credentialsId: 'aws-jenkins-creds']]) {
                    bat """
                    aws s3 cp target\\${SERVICE_NAME}.jar s3://${S3_BUCKET}/${SERVICE_NAME}.jar
                    """
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

                        # Convert password to secure string
                        \$pass = ConvertTo-SecureString '${EC2_PASSWORD}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential('Administrator', \$pass)
                        \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                        # Create remote session
                        \$session = New-PSSession -ComputerName ${EC2_HOST} -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption
                        if (-not \$session) { throw 'Failed to create PSSession to EC2.' }

                        Write-Output '--- Creating deployment directory if not exists ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$dir)
                            if (-not (Test-Path -Path \$dir)) {
                                New-Item -Path \$dir -ItemType Directory | Out-Null
                            }
                        } -ArgumentList '${DEPLOY_DIR}'

                        Write-Output '--- Stopping any running Java processes ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            Get-Process java -ErrorAction SilentlyContinue | Stop-Process -Force
                        }

                        Write-Output '--- Downloading JAR from S3 on EC2 ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$bucket, \$dest)
                            aws s3 cp s3://\$bucket/${SERVICE_NAME}.jar \$dest --region YOUR_AWS_REGION
                        } -ArgumentList '${S3_BUCKET}', '${DEPLOY_DIR}\\${SERVICE_NAME}.jar'

                        Write-Output '--- Starting Spring Boot service ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$jarPath, \$port)
                            Start-Process -FilePath 'java' -ArgumentList "-jar \$jarPath --server.port=\$port" -NoNewWindow
                        } -ArgumentList "${DEPLOY_DIR}\\${SERVICE_NAME}.jar", '${SERVICE_PORT}'

                        Write-Output '--- Closing remote session ---'
                        Remove-PSSession \$session
                    """
                }
            }
        }
    }
}
