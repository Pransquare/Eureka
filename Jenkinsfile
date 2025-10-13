pipeline {
    agent any // Windows agent not strictly required for this approach

    tools {
        jdk 'Java17'
        maven 'Maven3'
    }

    environment {
        S3_BUCKET = "my-deploy-bucket" // Replace with your S3 bucket name
        S3_KEY = "eureka-server/${BUILD_NUMBER}/eureka-server.jar"
        EC2_HOST = "13.53.193.215"
        DEPLOY_DIR = "C:\\Apps\\eureka-server"
        SERVICE_NAME = "eureka-server"
        SERVICE_PORT = "8761"
        AWS_CREDENTIALS_ID = "aws-credentials" // Jenkins AWS credentials ID
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
                withAWS(credentials: "${AWS_CREDENTIALS_ID}", region: 'ap-south-1') {
                    s3Upload(
                        bucket: "${S3_BUCKET}",
                        file: "${WORKSPACE}\\target\\${SERVICE_NAME}.jar",
                        path: "${S3_KEY}",
                        acl: 'private'
                    )
                }
            }
        }

        stage('Deploy on EC2') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'ec2-admin-creds',
                    usernameVariable: 'EC2_USER',
                    passwordVariable: 'EC2_PASSWORD'
                )]) {
                    powershell """
                        Write-Output '--- Connecting to EC2 ---'

                        # Convert password to secure string
                        \$pass = ConvertTo-SecureString '${EC2_PASSWORD}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential('Administrator', \$pass)
                        \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                        \$session = New-PSSession -ComputerName ${EC2_HOST} -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption
                        if (-not \$session) { throw 'Failed to create PSSession to EC2.' }

                        Write-Output '--- Creating deployment directory if not exists ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$dir)
                            if (-not (Test-Path -Path \$dir)) { New-Item -Path \$dir -ItemType Directory | Out-Null }
                        } -ArgumentList '${DEPLOY_DIR}'

                        Write-Output '--- Stopping any running Java processes ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            Get-Process java -ErrorAction SilentlyContinue | Stop-Process -Force
                        }

                        Write-Output '--- Downloading JAR from S3 ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$bucket, \$key, \$destination)
                            aws s3 cp "s3://\$bucket/\$key" \$destination
                        } -ArgumentList "${S3_BUCKET}", "${S3_KEY}", "${DEPLOY_DIR}\\${SERVICE_NAME}.jar"

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
