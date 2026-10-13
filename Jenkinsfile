pipeline {
    agent  {
        label 'AGENT-1'
    }
    environment { 
        appVersion = ''
        REGION = "us-east-1"
        ACC_ID = "211125615103"
        PROJECT = "roboshop"
        COMPONENT = "catalogue"
    }
    options {
        timeout(time: 30, unit: 'MINUTES') 
        disableConcurrentBuilds()
    }
    parameters {
        booleanParam(name: 'deploy', defaultValue: false, description: 'Toggle this value')
    }
    // Build
    stages {
        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    appVersion = packageJson.version
                    echo "Package version: ${appVersion}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                script {
                   sh """
                        npm install
                   """
                }
            }
        }
        stage('Unit Testing') {
            steps {
                script {
                   sh """
                        echo "unit tests"
                   """
                }
            }
        }
        /* stage('Sonar Scan') {
            environment {
                scannerHome = tool 'sonar-7.2'
            }
            steps {
                script {
                   // Sonar Server envrionment
                   withSonarQubeEnv(installationName: 'sonar-7.2') {
                         sh "${scannerHome}/bin/sonar-scanner"
                   }
                }
            }
        } */
        // Enable webhook in sonarqube server and wait for results
        /* stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true }
            }
        } */
        stage('Check Dependabot Alerts') {
            environment { 
                GITHUB_TOKEN = credentials('github-token')
            }
            steps {
                script {
                    // Fetch alerts from GitHub
                    def response = sh(
                        script: """
                            curl -s -H "Accept: application/vnd.github+json" \
                                 -H "Authorization: token ${GITHUB_TOKEN}" \
                                 https://api.github.com/repos/shivapaturi/catalogue/dependabot/alerts
                        """,
                        returnStdout: true
                    ).trim()

                    // Parse JSON
                    def json = readJSON text: response

                    // Filter alerts by severity
                    def criticalOrHigh = json.findAll { alert ->
                        def severity = alert?.security_advisory?.severity?.toLowerCase()
                        def state = alert?.state?.toLowerCase()
                        return (state == "open" && (severity == "critical" || severity == "high"))
                    }

                    if (criticalOrHigh.size() > 0) {
                        error "❌ Found ${criticalOrHigh.size()} HIGH/CRITICAL Dependabot alerts. Failing pipeline!"
                    } else {
                        echo "✅ No HIGH/CRITICAL Dependabot alerts found."
                    }
                }
            }
        }
        stage('Docker Build') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com
                            docker build -t ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion} .
                            docker push ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion}
                        """
                    }
                }
            }
        }
        stage('Check Scan Results') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        try {
                            // 1. Check scan status first to ensure results are ready
                            def statusJson = readJSON text: sh(
                                script: """
                                    aws ecr describe-image-scan-status \
                                    --repository-name ${PROJECT}/${COMPONENT} \
                                    --image-id imageTag=${appVersion} \
                                    --region ${REGION} \
                                    --output json
                                """,
                                returnStdout: true
                            ).trim()

                            def scanStatus = statusJson.imageScanStatus?.status
                            echo "ECR Image Scan Status: ${scanStatus}"

                            if (scanStatus == 'IN_PROGRESS') {
                                error("Image scan is still in progress. Please wait for completion before checking findings.")
                            } else if (scanStatus == 'FAILED') {
                                error("Image scan failed in ECR.")
                            } else if (scanStatus != 'COMPLETE') {
                                echo "Warning: Scan status is '${scanStatus}'. Proceeding with caution."
                            }

                            // 2. Fetch scan findings
                            def findingsOutput = sh(
                                script: """
                                    aws ecr describe-image-scan-findings \
                                    --repository-name ${PROJECT}/${COMPONENT} \
                                    --image-id imageTag=${appVersion} \
                                    --region ${REGION} \
                                    --output json
                                """,
                                returnStdout: true
                            ).trim()

                            def json = readJSON text: findingsOutput
                            def findingsList = json.imageScanFindings?.findings ?: []

                            def highCritical = findingsList.findAll {
                                it.severity == "HIGH" || it.severity == "CRITICAL"
                            }

                            if (highCritical.size() > 0) {
                                echo "❌ Found ${highCritical.size()} HIGH/CRITICAL vulnerabilities!"
                                currentBuild.result = 'FAILURE'
                                error("Build failed due to vulnerabilities")
                            } else {
                                echo "✅ No HIGH/CRITICAL vulnerabilities found."
                            }

                        } catch (Exception e) {
                            echo "⚠️ Error checking ECR scan results: ${e.getMessage()}"
                            currentBuild.result = 'FAILURE'
                            error("Vulnerability check failed due to an exception.")
                        }
                    }
                }
            }
        }
        stage('Trigger Deploy') {
            when{
                expression { params.deploy }
            }
            steps {
                script {
                    build job: 'catalogue-cd',
                    parameters: [
                        string(name: 'appVersion', value: "${appVersion}"),
                        string(name: 'deploy_to', value: 'dev')
                    ],
                    propagate: false,  // even SG fails VPC will not be effected
                    wait: false // VPC will not wait for SG pipeline completion
                }
            }
        }
        
    }

    post { 
        always { 
            echo 'I will always say Hello again!'
            deleteDir()
        }
        success { 
            echo 'Hello Success'
        }
        failure { 
            echo 'Hello Failure'
        }
    }
}