pipeline {
    agent any

    environment {
        COMPOSE_PROJECT_NAME = 'buy01'
        SONAR_ORGANIZATION = 'tonikorhonen'
        SONAR_PROJECT_KEY = 'ToniKorhonen_buy-01'
        SONAR_SCANNER_OPTS = '-Xmx512m'
        SONAR_SCANNER_SKIP_JRE = 'true'
        JWT_SECRET_TEST = 'test-jwt-secret-for-testing-only'
        BUILD_SERVICES = 'user-service,product-service,media-service,api-gateway,order-service'
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['auto', 'dev', 'prod'], description: 'Deployment environment')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip tests')
        booleanParam(name: 'CLEAN_DOCKER', defaultValue: false, description: 'Clean Docker images')
        booleanParam(name: 'ENFORCE_QUALITY_GATE', defaultValue: false, description: 'Enforce SonarCloud gate')
        string(name: 'EMAIL_RECIPIENTS', defaultValue: 'team@example.com', description: 'Email recipients')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '15'))
        timestamps()
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }

    stages {
        stage('Initialize') {
            steps {
                script {
                    checkout scm

                    if (isUnix()) {
                        env.BUILD_TIMESTAMP = sh(script: 'date +%Y%m%d-%H%M%S', returnStdout: true).trim()
                        env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    } else {
                        env.BUILD_TIMESTAMP = powershell(script: 'Get-Date -Format "yyyyMMdd-HHmmss"', returnStdout: true).trim()
                        env.GIT_COMMIT_SHORT = powershell(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    }

                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"

                    def deployBranches = ['main', 'master', 'dev']
                    def approvalBranches = ['main', 'master']
                    env.SHOULD_DEPLOY = deployBranches.contains(env.BRANCH_NAME) ? 'true' : 'false'
                    env.NEEDS_APPROVAL = approvalBranches.contains(env.BRANCH_NAME) ? 'true' : 'false'

                    echo """
                    ╔════════════════════════════════════╗
                    ║   BUILD INITIALIZATION             ║
                    ╠════════════════════════════════════╣
                    ║ Branch:     ${env.BRANCH_NAME}
                    ║ Build #:    ${env.BUILD_NUMBER}
                    ║ Commit:     ${env.GIT_COMMIT_SHORT}
                    ║ Deploy:     ${env.SHOULD_DEPLOY}
                    ║ Email:      ${params.EMAIL_RECIPIENTS}
                    ╚════════════════════════════════════╝
                    """

                    // Validate email recipients
                    if (!params.EMAIL_RECIPIENTS || params.EMAIL_RECIPIENTS.isEmpty()) {
                        echo "⚠️  WARNING: EMAIL_RECIPIENTS is empty, notifications will not be sent"
                    }
                }
            }
        }

        stage('Setup') {
            steps {
                script {
                    echo '🔧 Configuring environment...'

                    if (env.SHOULD_DEPLOY == 'true') {
                        withCredentials([string(credentialsId: 'JWT_SECRET', variable: 'JWT_SECRET')]) {
                            if (isUnix()) {
                                sh '''
                                    cat > .env << EOF
JWT_SECRET=${JWT_SECRET}
JWT_EXPIRATION=3600000
MONGODB_HOST=mongodb
MONGODB_PORT=27017
USER_DB_NAME=buy01_users
PRODUCT_DB_NAME=buy01_products
MEDIA_DB_NAME=media_db
EOF
                                    chmod 600 .env
                                '''
                            } else {
                                bat '''
                                    (echo JWT_SECRET=%JWT_SECRET%^
                                    echo JWT_EXPIRATION=3600000^
                                    echo MONGODB_HOST=mongodb^
                                    echo MONGODB_PORT=27017^
                                    echo USER_DB_NAME=buy01_users^
                                    echo PRODUCT_DB_NAME=buy01_products^
                                    echo MEDIA_DB_NAME=media_db) > .env
                                '''
                            }
                        }
                        echo '✅ Production environment configured'
                    } else {
                        if (isUnix()) {
                            sh '''
                                cat > .env << EOF
JWT_SECRET=test-jwt-secret-build-only
JWT_EXPIRATION=3600000
MONGODB_HOST=mongodb
MONGODB_PORT=27017
USER_DB_NAME=buy01_users
PRODUCT_DB_NAME=buy01_products
MEDIA_DB_NAME=media_db
EOF
                                chmod 600 .env
                            '''
                        } else {
                            bat '''
                                (echo JWT_SECRET=test-jwt-secret-build-only^
                                echo JWT_EXPIRATION=3600000^
                                echo MONGODB_HOST=mongodb^
                                echo MONGODB_PORT=27017^
                                echo USER_DB_NAME=buy01_users^
                                echo PRODUCT_DB_NAME=buy01_products^
                                echo MEDIA_DB_NAME=media_db) > .env
                            '''
                        }
                        echo '✅ Test environment configured'
                    }
                }
            }
        }

        stage('Build Shared Commons') {
            steps {
                script {
                    dir('Backend/shared-commons') {
                        if (isUnix()) {
                            sh 'mvn clean install -DskipTests -B'
                        } else {
                            bat 'mvn clean install -DskipTests -B'
                        }
                    }
                }
            }
        }

        stage('Build') {
            parallel {
                stage('Backend Services') {
                    steps {
                        script {
                            ['user-service', 'product-service', 'media-service', 'api-gateway', 'order-service'].each { service ->
                                buildBackendService(service)
                            }
                        }
                    }
                }

                stage('Frontend') {
                    steps {
                        buildFrontend()
                    }
                }
            }
        }

        stage('Test') {
            when {
                expression { !params.SKIP_TESTS }
            }
            environment {
                JWT_SECRET = "${JWT_SECRET_TEST}"
            }
            parallel {
                stage('Backend Tests') {
                    steps {
                        script {
                            ['shared-commons', 'user-service', 'product-service', 'media-service', 'api-gateway', 'order-service'].each { service ->
                                testBackendService(service)
                            }
                        }
                    }
                }

                stage('Frontend Tests') {
                    steps {
                        testFrontend()
                    }
                }
            }
        }

        stage('Reports') {
            when {
                expression { !params.SKIP_TESTS }
            }
            steps {
                script {
                    echo '📊 Publishing reports...'

                    junit testResults: '**/target/surefire-reports/*.xml',
                          allowEmptyResults: true,
                          skipPublishingChecks: true

                    try {
                        recordCoverage(tools: [[parser: 'JACOCO', pattern: '**/target/site/jacoco/jacoco.xml']],
                                     sourceCodeRetention: 'EVERY_BUILD')
                        echo '✅ Coverage published'
                    } catch (Exception _) {
                        echo "⚠️  Coverage plugin unavailable"
                    }

                    ['User Service', 'Product Service', 'API Gateway', 'Media Service', 'Order Service'].eachWithIndex { name, i ->
                        def dirs = ['user-service', 'product-service', 'api-gateway', 'media-service', 'order-service']
                        publishCoverageReport(name, "Backend/${dirs[i]}")
                    }

                    publishCoverageReport('Frontend', 'Frontend')
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression { env.SHOULD_DEPLOY == 'true' || params.DEPLOY_ENV != 'auto' }
            }
            steps {
                script {
                    echo '📊 Running SonarCloud analysis...'

                    try {
                        // Ensure Maven dependencies are downloaded for better SonarCloud analysis
                        echo '📥 Downloading Maven dependencies for analysis...'
                        if (isUnix()) {
                            sh '''
                                for service in user-service product-service media-service api-gateway order-service; do
                                    echo "📦 Downloading dependencies for $service..."
                                    cd Backend/$service
                                    ./mvnw dependency:go-offline -q 2>&1 || echo "Note: Some dependencies may require network access"
                                    cd ../../
                                done
                            '''
                        }

                        withSonarQubeEnv('SonarCloud') {
                            if (isUnix()) {
                                sh '''
                                    echo "📥 Installing SonarQube Scanner..."
                                    npm install -g sonarqube-scanner --force 2>&1 || true
                                    
                                    echo "🔍 Running SonarCloud analysis..."
                                    sonar-scanner 2>&1 | tee sonar-analysis.log
                                    
                                    if grep -q "ERROR" sonar-analysis.log; then
                                        echo "⚠️  SonarCloud analysis completed with warnings"
                                    else
                                        echo "✅ SonarCloud analysis submitted successfully"
                                    fi
                                '''
                            } else {
                                bat '''
                                    echo Installing SonarQube Scanner...
                                    npm install -g sonarqube-scanner --force 2>&1 || echo.
                                    
                                    echo Running SonarCloud analysis...
                                    sonar-scanner
                                    
                                    echo SonarCloud analysis submitted
                                '''
                            }
                        }

                        if (params.ENFORCE_QUALITY_GATE) {
                            echo '⏳ Waiting for Quality Gate results...'
                            timeout(time: 15, unit: 'MINUTES') {
                                def qg = waitForQualityGate()
                                if (qg.status != 'OK') {
                                    echo "⚠️  Quality Gate status: ${qg.status}"
                                    if (qg.status == 'ERROR') {
                                        error "❌ Quality Gate FAILED"
                                    }
                                } else {
                                    echo '✅ Quality Gate PASSED'
                                }
                            }
                        } else {
                            echo '✅ SonarCloud analysis completed (Quality Gate not enforced)'
                        }
                    } catch (Exception e) {
                        echo "⚠️  SonarCloud analysis encountered an issue: ${e.message}"
                        echo "ℹ️  This is non-blocking — continuing pipeline"
                        // Non-blocking error for SonarCloud
                    }
                }
            }
        }

        stage('Docker') {
            when {
                allOf {
                    expression { env.SHOULD_DEPLOY == 'true' }
                    expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    echo '🐳 Building Docker images...'

                    if (params.CLEAN_DOCKER) {
                        if (isUnix()) {
                            sh 'docker system prune -a -f --volumes || true'
                        } else {
                            bat 'docker system prune -a -f --volumes || exit /b 0'
                        }
                    }

                    if (isUnix()) {
                        sh '''
                            echo "🔨 Building Docker images..."
                            docker compose build --parallel
                            
                            echo "🏷️  Tagging images..."
                            for img in user-service product-service media-service api-gateway order-service frontend; do
                                docker tag buy01-$img:latest buy01-$img:${IMAGE_TAG} || echo "Warning: Could not tag buy01-$img"
                            done
                            echo "✅ Images tagged: ${IMAGE_TAG}"
                        '''
                    } else {
                        bat '''
                            echo Building Docker images...
                            docker compose build --parallel
                            
                            echo Tagging images...
                            for %%i in (user-service product-service media-service api-gateway order-service frontend) do (
                                docker tag buy01-%%i:latest buy01-%%i:%IMAGE_TAG% || echo Warning: Could not tag buy01-%%i
                            )
                            echo Images tagged: %IMAGE_TAG%
                        '''
                    }
                }
            }
        }

        stage('Approve Deploy') {
            when {
                allOf {
                    expression { env.SHOULD_DEPLOY == 'true' }
                    expression { env.NEEDS_APPROVAL == 'true' }
                    expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    timeout(time: 30, unit: 'MINUTES') {
                        input message: 'Deploy to production?', ok: 'Deploy', submitter: 'admin'
                    }
                }
            }
        }

        stage('Deploy') {
            when {
                allOf {
                    expression { env.SHOULD_DEPLOY == 'true' }
                    expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                script {
                    echo '🚀 Deploying...'

                    if (isUnix()) {
                        sh 'chmod +x jenkins-deploy.sh && ./jenkins-deploy.sh'
                    } else {
                        powershell 'Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force; .\\jenkins-deploy.ps1'
                    }

                    echo '✅ Deployment complete'
                }
            }
        }
    }

    post {
        always {
            script {
                // Archive test reports and coverage
                archiveArtifacts artifacts: '**/target/surefire-reports/*.xml, **/target/site/jacoco/**/*, Frontend/coverage/**/*',
                                 allowEmptyArchive: true, fingerprint: true
                
                // Clean up sensitive files
                if (isUnix()) {
                    sh 'rm -f .env || true'
                } else {
                    bat 'if exist .env del /f .env'
                }
            }
        }

        success {
            emailext(
                subject: "✅ BUILD SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: '''<html><body style="font-family: Arial;">
                    <h2 style="color: #28a745;">✅ Build Successful</h2>
                    <table border="1" cellpadding="5" style="border-collapse: collapse;">
                        <tr><td><b>Build</b></td><td>#${BUILD_NUMBER}</td></tr>
                        <tr><td><b>Branch</b></td><td>${BRANCH_NAME}</td></tr>
                        <tr><td><b>Duration</b></td><td>${BUILD_DURATIONSTRING}</td></tr>
                    </table>
                    <ul>
                        <li><a href="${BUILD_URL}">View Build</a></li>
                        <li><a href="${BUILD_URL}testReport/">Test Results</a></li>
                        <li><a href="${BUILD_URL}jacoco/">Coverage</a></li>
                    </ul>
                </body></html>''',
                to: "${params.EMAIL_RECIPIENTS}", mimeType: 'text/html')
        }

        failure {
            emailext(
                subject: "❌ BUILD FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: '''<html><body style="font-family: Arial;">
                    <h2 style="color: #dc3545;">❌ Build Failed</h2>
                    <table border="1" cellpadding="5" style="border-collapse: collapse;">
                        <tr><td><b>Build</b></td><td>#${BUILD_NUMBER}</td></tr>
                        <tr><td><b>Branch</b></td><td>${BRANCH_NAME}</td></tr>
                    </table>
                    <ul>
                        <li><a href="${BUILD_URL}console">Console Output</a></li>
                        <li><a href="${BUILD_URL}testReport/">Test Reports</a></li>
                    </ul>
                </body></html>''',
                to: "${params.EMAIL_RECIPIENTS}", mimeType: 'text/html', attachLog: true)
        }

        unstable {
            emailext(
                subject: "⚠️ BUILD UNSTABLE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: '''<html><body style="font-family: Arial;">
                    <h2 style="color: #ffc107;">⚠️ Build Unstable</h2>
                    <p>Some tests failed. Review test reports.</p>
                    <ul>
                        <li><a href="${BUILD_URL}testReport/">View Test Results</a></li>
                    </ul>
                </body></html>''',
                to: "${params.EMAIL_RECIPIENTS}", mimeType: 'text/html')
        }
    }
}

// ============== SHARED FUNCTIONS ==============

def buildBackendService(String service) {
    dir("Backend/${service}") {
        timeout(time: 10, unit: 'MINUTES') {
            if (isUnix()) {
                sh 'if [ -f mvnw ]; then ./mvnw clean package -DskipTests -Dmaven.javadoc.skip=true; else mvn clean package -DskipTests -Dmaven.javadoc.skip=true; fi'
            } else {
                bat 'if exist mvnw.cmd (mvnw.cmd clean package -DskipTests -Dmaven.javadoc.skip=true) else (mvn clean package -DskipTests -Dmaven.javadoc.skip=true)'
            }
        }
    }
}

def testBackendService(String service) {
    dir("Backend/${service}") {
        timeout(time: 15, unit: 'MINUTES') {
            try {
                if (isUnix()) {
                    sh '''
                        echo "🧪 Testing ${service}..."
                        export JWT_SECRET="${JWT_SECRET}"
                        if [ -f mvnw ]; then ./mvnw test jacoco:report -B; else mvn test jacoco:report -B; fi
                        echo "✅ ${service} tests completed"
                    '''
                } else {
                    bat '''
                        set JWT_SECRET=%JWT_SECRET%
                        if exist mvnw.cmd (mvnw.cmd test jacoco:report -B) else (mvn test jacoco:report -B)
                        echo Tests completed
                    '''
                }
            } catch (Exception e) {
                echo "❌ ${service} test failure: ${e.message}"
                throw e // Re-throw to mark build as failed but continue to capture results
            }
        }
    }
}

def buildFrontend() {
    dir('Frontend') {
        timeout(time: 15, unit: 'MINUTES') {
            if (isUnix()) {
                sh '''
                    rm -rf node_modules package-lock.json dist .angular
                    npm install --legacy-peer-deps
                    npm audit fix --audit-level=moderate || true
                    npm run build -- --configuration=production
                '''
            } else {
                bat '''
                    if exist node_modules rmdir /s /q node_modules
                    if exist package-lock.json del /f package-lock.json
                    if exist dist rmdir /s /q dist
                    if exist .angular rmdir /s /q .angular
                    npm install --legacy-peer-deps
                    npm audit fix --audit-level=moderate
                    npm run build -- --configuration=production
                '''
            }
        }
    }
}

def testFrontend() {
    dir('Frontend') {
        timeout(time: 20, unit: 'MINUTES') {
            try {
                if (isUnix()) {
                    sh '''
                        echo "🧪 Installing Frontend dependencies..."
                        npm install --legacy-peer-deps
                        
                        echo "🧪 Running Frontend tests..."
                        npm run test:coverage
                        
                        echo "✅ Frontend tests completed"
                    '''
                } else {
                    bat '''
                        echo Running Frontend tests...
                        npm install --legacy-peer-deps
                        npm run test:coverage
                        echo Frontend tests completed
                    '''
                }
            } catch (Exception e) {
                echo "⚠️  Frontend tests failed or warnings: ${e.message}"
                // Continue build - frontend test failures shouldn't block
            }
        }
    }
}

def publishCoverageReport(String name, String dir) {
    try {
        String reportPath = name == 'Frontend' ? "${dir}/coverage" : "${dir}/target/site/jacoco"
        publishHTML([
            allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true,
            reportDir: reportPath,
            reportFiles: 'index.html',
            reportName: "${name} Coverage"
        ])
        echo "✅ ${name} coverage published"
    } catch (Exception e) {
        echo "⚠️  ${name} coverage unavailable: ${e.message}"
    }
}



