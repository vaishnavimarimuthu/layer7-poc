pipeline {
    agent any
 
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'test', 'prod'],
            description: 'Select target Layer7 environment'
        )
    }
 
    environment {
        BUNDLE_FILE = 'bundles/Configuration-Cache-Demo.bundle'
        GMU_HOME = 'C:\\gmu'
    }
 
    stages {
 
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
 
        stage('Load Environment Configuration') {
            steps {
                script {
                    def envConfig = readYaml file: "config/${params.ENVIRONMENT}.yaml"
 
                    env.GATEWAY_HOST = envConfig.gateway.host
                    env.GATEWAY_PORT = envConfig.gateway.port.toString()
                    env.GATEWAY_PROTOCOL = envConfig.gateway.protocol
 
                    echo "Target Environment : ${params.ENVIRONMENT}"
                    echo "Gateway Host       : ${env.GATEWAY_HOST}"
                    echo "Gateway Port       : ${env.GATEWAY_PORT}"
                }
            }
        }
 
        stage('Validate Bundle') {
            steps {
                script {
                    if (!fileExists(env.BUNDLE_FILE)) {
                        error "Bundle file not found: ${env.BUNDLE_FILE}"
                    }
 
                    echo "Bundle found: ${env.BUNDLE_FILE}"
                }
            }
        }
 
        stage('Validate GMU') {
            steps {
                // Changed from sh to bat and updated syntax for Windows CLI
                bat '''
                    @echo off
                    echo Checking GMU installation...
 
                    if not exist "%GMU_HOME%\\GatewayMigrationUtility.bat" (
                        echo GMU not found at %GMU_HOME%
                        exit /b 1
                    )
 
                    echo GMU installation found.
                '''
            }
        }
 
        stage('Deploy to Layer7') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'layer7-gateway-credentials',
                        usernameVariable: 'GATEWAY_USERNAME',
                        passwordVariable: 'GATEWAY_PASSWORD'
                    )
                ]) {
 
                    // Changed from sh to bat and converted line continuation to Windows caret (^)
                    bat '''
                        @echo off
                        echo Deploying bundle to Layer7 Gateway...
 
                        call "%GMU_HOME%\\GatewayMigrationUtility.bat" ^
                            migrateIn ^
                            --host %GATEWAY_HOST% ^
                            --port %GATEWAY_PORT% ^
                            --username "%GATEWAY_USERNAME%" ^
                            --password "%GATEWAY_PASSWORD%" ^
                            --bundle "%BUNDLE_FILE%"
 
                        if %ERRORLEVEL% neq 0 (
                            echo Deployment failed with error code %ERRORLEVEL%.
                            exit /b %ERRORLEVEL%
                        )
 
                        echo Deployment completed.
                    '''
                }
            }
        }
 
        stage('Deployment Verification') {
            steps {
                echo "GMU deployment completed successfully."
            }
        }
    }
 
    post {
 
        success {
            echo """
            ==========================================
            Layer7 Deployment SUCCESS
            Environment : ${params.ENVIRONMENT}
            Bundle      : ${env.BUNDLE_FILE}
            ==========================================
            """
        }
 
        failure {
            echo """
            ==========================================
            Layer7 Deployment FAILED
            Environment : ${params.ENVIRONMENT}
            ==========================================
            """
        }
 
        always {
            cleanWs()
        }
    }
}
