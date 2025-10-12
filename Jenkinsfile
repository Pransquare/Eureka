pipeline {
    agent any // Ensure this is a Windows agent with PowerShell installed

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
                        Write-Output '--- Starting deployment to EC2 ---'

                        # Convert password to secure string
                        \$pass = ConvertTo-SecureString '${EC2_PASSWORD}' -AsPlainText -Force
                        \$cred = New-Object System.Management.Automation.PSCredential('Administrator', \$pass)

                        # Skip SSL certificate validation for self-signed certs
                        \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                        # Create remote session using Basic authentication
                        \$session = New-PSSession -ComputerName ${EC2_HOST} -UseSSL -Credential \$cred -Authentication Basic -SessionOption \$sessOption
                        if (-not \$session) { throw 'Failed to create PSSession to EC2.' }

                        # Pick the latest JAR in the target folder
                        Write-Output '--- Picking latest JAR from target folder ---'
                        \$jar = Get-ChildItem "target\\${SERVICE_NAME}*.jar" | Sort-Object LastWriteTime -Descending | Select-Object -First 1
                        if (-not \$jar) { throw "No JAR found in target folder!" }
                        Write-Output "Latest JAR: \$($jar.FullName)"

                        # Create deployment directory on EC2
                        Write-Output '--- Creating deployment directory if not exists ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$dir)
                            if (-not (Test-Path -Path \$dir)) { New-Item -Path \$dir -ItemType Directory | Out-Null }
                        } -ArgumentList '${DEPLOY_DIR}'

                        # Stop existing Java processes on EC2
                        Write-Output '--- Stopping existing Java processes ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            Get-Process java -ErrorAction SilentlyContinue | Stop-Process -Force
                        }

                        # Copy the JAR to EC2
                        Write-Output '--- Copying JAR to EC2 via WinRM ---'
                        Copy-Item -Path \$jar.FullName -Destination "${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -ToSession \$session -Force

                        # Start Spring Boot service on EC2
                        Write-Output '--- Starting Spring Boot service ---'
                        Invoke-Command -Session \$session -ScriptBlock {
                            param(\$jarPath, \$port)
                            Start-Process -FilePath 'java' -ArgumentList "-jar \$jarPath --server.port=\$port" -NoNewWindow
                        } -ArgumentList "${DEPLOY_DIR}\\${SERVICE_NAME}.jar", '${SERVICE_PORT}'

                        # Close remote session
                        Write-Output '--- Closing remote session ---'
                        Remove-PSSession \$session

                        Write-Output '--- Deployment completed successfully ---'
                    """
                }
            }
        }
    }
}
