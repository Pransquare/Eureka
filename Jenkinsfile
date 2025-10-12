pipeline {
    agent any // Jenkins agent should have PowerShell available (Windows agent recommended)

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

                # Convert Jenkins password to secure string
                \$pass = ConvertTo-SecureString '${EC2_PASSWORD}' -AsPlainText -Force
                \$cred = New-Object System.Management.Automation.PSCredential ('${EC2_USER}', \$pass)

                # Skip SSL certificate validation for self-signed certificate
                \$sessOption = New-PSSessionOption -SkipCACheck -SkipCNCheck

                # Create remote session
                try {
                    \$session = New-PSSession -ComputerName ${EC2_HOST} -UseSSL -Credential \$cred -SessionOption \$sessOption
                    Write-Output '--- Connected to EC2 successfully ---'
                } catch {
                    Write-Error 'Failed to create PSSession to EC2. Check credentials, WinRM, or SSL.'
                    throw
                }

                # Create deployment directory if it doesn't exist
                Invoke-Command -Session \$session -ScriptBlock {
                    param(\$dir)
                    if (-not (Test-Path -Path \$dir)) {
                        New-Item -Path \$dir -ItemType Directory | Out-Null
                        Write-Output "Created deployment directory: \$dir"
                    } else {
                        Write-Output "Deployment directory already exists: \$dir"
                    }
                } -ArgumentList '${DEPLOY_DIR}'

                # Stop only existing eureka-server Java processes
                Invoke-Command -Session \$session -ScriptBlock {
                    \$javaProcesses = Get-Process java -ErrorAction SilentlyContinue | Where-Object { \$_.Path -like '*${SERVICE_NAME}*' }
                    if (\$javaProcesses) {
                        \$javaProcesses | Stop-Process -Force
                        Write-Output 'Stopped existing eureka-server Java processes.'
                    } else {
                        Write-Output 'No eureka-server Java processes running.'
                    }
                }

                # Copy JAR to EC2
                Copy-Item -Path "target\\${SERVICE_NAME}.jar" -Destination "${DEPLOY_DIR}\\${SERVICE_NAME}.jar" -ToSession \$session -Force
                Write-Output 'Copied JAR to EC2.'

                # Start Spring Boot service
                Invoke-Command -Session \$session -ScriptBlock {
                    param(\$jarPath, \$port)
                    Start-Process -FilePath 'java' -ArgumentList "-jar \$jarPath --server.port=\$port" -NoNewWindow
                    Write-Output "Started Spring Boot service on port \$port"
                } -ArgumentList "${DEPLOY_DIR}\\${SERVICE_NAME}.jar", '${SERVICE_PORT}'

                # Close session
                Remove-PSSession \$session
                Write-Output '--- Deployment completed and remote session closed ---'
            """
        }
    }
}

    }
}
