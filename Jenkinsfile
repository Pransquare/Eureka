pipeline {    
    agent any  // Ensure this is a Windows agent with PowerShell installed    

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
                    powershell '''
                        Write-Output "=== Starting deployment to EC2 (${env:EC2_HOST}) ==="

                        try {
                            # Convert password to secure string
                            $securePass = ConvertTo-SecureString $env:EC2_PASSWORD -AsPlainText -Force
                            $cred = New-Object System.Management.Automation.PSCredential ($env:EC2_USER, $securePass)
                            $sessionOptions = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck

                            Write-Output "--- Creating remote PowerShell session ---"
                            $session = New-PSSession -ComputerName $env:EC2_HOST -UseSSL -Credential $cred -Authentication Basic -SessionOption $sessionOptions
                            if (-not $session) { throw "❌ Failed to create PSSession to EC2 instance." }

                            Write-Output "--- Ensuring deployment directory exists ---"
                            Invoke-Command -Session $session -ScriptBlock {
                                param($dir)
                                if (-not (Test-Path -Path $dir)) {
                                    New-Item -Path $dir -ItemType Directory | Out-Null
                                    Write-Output "Created directory: $dir"
                                } else {
                                    Write-Output "Directory already exists: $dir"
                                }
                            } -ArgumentList $env:DEPLOY_DIR

                            Write-Output "--- Stopping any running Java processes ---"
                            Invoke-Command -Session $session -ScriptBlock {
                                $procs = Get-Process java -ErrorAction SilentlyContinue
                                if ($procs) {
                                    $procs | Stop-Process -Force
                                    Write-Output "Stopped existing Java processes."
                                } else {
                                    Write-Output "No running Java processes found."
                                }
                            }

                            Write-Output "--- Copying JAR file to EC2 ---"
                            Copy-Item -ToSession $session `
                                -Path "$env:WORKSPACE\\target\\$env:SERVICE_NAME.jar" `
                                -Destination "$env:DEPLOY_DIR\\$env:SERVICE_NAME.jar" `
                                -Force

                            Write-Output "--- Starting Spring Boot service ---"
                            Invoke-Command -Session $session -ScriptBlock {
                                param($jarPath, $port)
                                $logPath = Join-Path (Split-Path $jarPath) "eureka.log"
                                Write-Output "Starting JAR: $jarPath on port $port"
                                Start-Process "java" "-jar `"$jarPath`" --server.port=$port" -RedirectStandardOutput $logPath -RedirectStandardError $logPath -WindowStyle Hidden
                                Write-Output "Application started. Logs: $logPath"
                            } -ArgumentList "$env:DEPLOY_DIR\\$env:SERVICE_NAME.jar", $env:SERVICE_PORT

                            Write-Output "--- Closing remote session ---"
                            Remove-PSSession $session

                            Write-Output "✅ Deployment completed successfully."
                        }
                        catch {
                            Write-Output "❌ Deployment failed: $($_.Exception.Message)"
                            exit 1
                        }
                    '''
                }            
            }        
        }    
    }
}
