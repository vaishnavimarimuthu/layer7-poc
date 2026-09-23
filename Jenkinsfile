pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'test', 'prod'], description: 'Select target Layer7 environment')
        string(name: 'RELEASE', defaultValue: 'R1', description: 'Release to run: R1, R2, R3, R4, or ALL. Case-insensitive.')
        string(name: 'APP_FILTER', defaultValue: '', description: 'Optional: comma/space-separated API names, e.g. "Configuration-Cache-Demo,HRT-API". Blank = all APIs in the release.')
    }

    environment {
        GMU_HOME = 'C:\\gmu'
        JAVA_HOME = 'C:\\Program Files\\Java\\jdk-17.0.18'
        PATH = "${env.JAVA_HOME}\\bin;${env.PATH}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Load Environment & Manifest Config') {
            steps {
                script {
                    def envConfig = readYaml file: "config/${params.ENVIRONMENT}.yaml"
                    env.GATEWAY_HOST = envConfig.gateway.host
                    env.GATEWAY_PORT = envConfig.gateway.port.toString()
                    env.GATEWAY_PROTOCOL = envConfig.gateway.protocol

                    echo "Target Environment : ${params.ENVIRONMENT}"
                    echo "Gateway Host       : ${env.GATEWAY_HOST}"
                    echo "Gateway Port       : ${env.GATEWAY_PORT}"

                    def releaseInput = params.RELEASE?.trim()?.toUpperCase() ?: 'R1'
                    def releasesToRun = []

                    if (releaseInput == 'ALL') {
                        releasesToRun = ['R1', 'R2', 'R3', 'R4'].findAll { fileExists("releases/${it}/manifest.yaml") }
                    } else {
                        releasesToRun = releaseInput.split('[,\\s]+')
                            .collect { it.trim() }
                            .findAll { it && fileExists("releases/${it}/manifest.yaml") }
                    }

                    if (releasesToRun.isEmpty()) {
                        error "No valid releases found for RELEASE=${params.RELEASE}"
                    }

                    def appFilter = params.APP_FILTER?.trim()
                        ? params.APP_FILTER.split('[,\\s]+').collect { it.trim() }.findAll { it }
                        : []

                    logReleaseApiMap = [:]

                    releasesToRun.each { release ->
                        def manifestPath = "releases/${release}/manifest.yaml"
                        def manifest = readYaml file: manifestPath
                        def apps = (manifest.services ?: []).collect { it.toString() }

                        if (appFilter) {
                            def filterLower = appFilter.collect { it.toLowerCase() }
                            apps = apps.findAll { filterLower.contains(it.toLowerCase()) }
                            echo "Release ${release}: filtered to [${apps.join(', ')}]"
                        }

                        if (apps.isEmpty()) {
                            error "No valid APIs found for Release ${release} with current APP_FILTER."
                        }

                        logReleaseApiMap[release] = apps
                        echo "Final APIs for Release ${release}: [${apps.join(', ')}]"
                    }
                }
            }
        }

        stage('Validate API Bundles') {
            steps {
                script {
                    logReleaseApiMap.each { release, apps ->
                        apps.each { app ->
                            String apiDir = "apis/${app}"
                            String bundlePath = "${apiDir}/${app}.xml"

                            if (!fileExists(apiDir)) {
                                error "API directory not found: ${apiDir}"
                            }
                            if (!fileExists(bundlePath)) {
                                error "GMU bundle XML not found: ${bundlePath}"
                            }

                            echo "Validated GMU bundle: ${bundlePath}"
                        }
                    }
                }
            }
        }

        stage('Validate GMU') {
            steps {
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

        stage('Prepare GMU Update Mappings') {
            steps {
                script {
                    logReleaseApiMap.each { release, apps ->
                        apps.each { app ->
                            String bundlePath = "apis/${app}/${app}.xml"

                            echo "Preparing NewOrUpdate mappings for ${bundlePath}"

                            ['POLICY', 'SERVICE', 'ENCAPSULATED_ASSERTION'].each { entityType ->
                                int rc = bat(returnStatus: true, script: """
                                    @echo off
                                    call "%GMU_HOME%\\GatewayMigrationUtility.bat" manageMappings -b "${bundlePath}" -t ${entityType} -a NewOrUpdate
                                """)
                                if (rc != 0) {
                                    error "manageMappings failed for ${entityType} in ${app}. Exit code: ${rc}"
                                }
                            }

                            echo "NewOrUpdate mappings prepared for ${app}"
                        }
                    }
                }
            }
        }

        stage('Test Layer7 Migration') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'layer7-gateway-credentials', usernameVariable: 'GATEWAY_USERNAME', passwordVariable: 'GATEWAY_PASSWORD')]) {
                    script {
                        logReleaseApiMap.each { release, apps ->
                            apps.each { app ->
                                String bundlePath = "apis/${app}/${app}.xml"
                                String safeApp = app.replaceAll('[^A-Za-z0-9_.-]', '_')

                                echo "Testing bundle: ${bundlePath}"

                                int rc = bat(returnStatus: true, script: """
                                    @echo off
                                    set "PATH=%JAVA_HOME%\\bin;%PATH%"
                                    call "%GMU_HOME%\\GatewayMigrationUtility.bat" migrateIn ^
                                        -h "%GATEWAY_HOST%" ^
                                        -p "%GATEWAY_PORT%" ^
                                        -u "%GATEWAY_USERNAME%" ^
                                        --plaintextPassword "%GATEWAY_PASSWORD%" ^
                                        --bundle "${bundlePath}" ^
                                        --plaintextEncryptionPassphrase "%GATEWAY_PASSWORD%" ^
                                        --results "results-${safeApp}.xml" ^
                                        --trustCertificate ^
                                        --trustHostname ^
                                        --test
                                """)

                                if (rc != 0) {
                                    error "GMU test migration failed for ${app}. Exit code: ${rc}"
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Layer7') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'layer7-gateway-credentials', usernameVariable: 'GATEWAY_USERNAME', passwordVariable: 'GATEWAY_PASSWORD')]) {
                    script {
                        logReleaseApiMap.each { release, apps ->
                            apps.each { app ->
                                String bundlePath = "apis/${app}/${app}.xml"
                                String safeApp = app.replaceAll('[^A-Za-z0-9_.-]', '_')

                                echo "Deploying ${bundlePath} to Layer7 Gateway..."

                                int rc = bat(returnStatus: true, script: """
                                    @echo off
                                    set "PATH=%JAVA_HOME%\\bin;%PATH%"
                                    call "%GMU_HOME%\\GatewayMigrationUtility.bat" migrateIn ^
                                        --host "%GATEWAY_HOST%" ^
                                        --port "%GATEWAY_PORT%" ^
                                        --username "%GATEWAY_USERNAME%" ^
                                        --plaintextPassword "%GATEWAY_PASSWORD%" ^
                                        --bundle "${bundlePath}" ^
                                        --plaintextEncryptionPassphrase "%GATEWAY_PASSWORD%" ^
                                        --results "gmu-results-${safeApp}.xml" ^
                                        --trustCertificate ^
                                        --trustHostname
                                """)

                                if (rc != 0) {
                                    error "Deployment failed for ${app}. Exit code: ${rc}"
                                }
                                echo "Successfully deployed: ${app}"
                            }
                        }
                    }
                }
            }
        }

        stage('Deployment Verification') {
            steps {
                echo 'All specified GMU deployments completed successfully.'
            }
        }
    }

    post {
        success {
            script {
                echo "Layer7 Deployment SUCCESS - Environment: ${params.ENVIRONMENT}"
                logReleaseApiMap.each { release, apps ->
                    echo "Release ${release}: ${apps.join(', ')}"
                }
            }
        }
        failure {
            echo "Layer7 Deployment FAILED - Environment: ${params.ENVIRONMENT}"
        }
        always {
            archiveArtifacts artifacts: 'results-*.xml, gmu-results-*.xml', allowEmptyArchive: true
            cleanWs()
        }
    }
}
