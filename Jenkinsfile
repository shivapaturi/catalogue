pipeline {
    agent {
        label 'AGENT-1'
    }

    environment {
        APP_VERSION = ''
        REGION      = 'us-east-1'
        ACC_ID      = '211125615103'
        PROJECT     = 'roboshop'
        COMPONENT   = 'catalogue'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(
            name: 'deploy',
            defaultValue: false,
            description: 'Deploy catalogue after security checks pass'
        )
    }

    stages {

        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'

                    env.APP_VERSION = packageJson.version?.toString()?.trim()

                    if (!env.APP_VERSION) {
                        error '❌ Version not found in package.json'
                    }

                    echo "Package version: ${env.APP_VERSION}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    set -e
                    npm install
                '''
            }
        }

        stage('Unit Testing') {
            steps {
                sh '''
                    echo "unit tests"
                '''
            }
        }

        stage('Check Dependabot Alerts') {
            environment {
                GITHUB_TOKEN = credentials('github-token')
            }

            steps {
                script {
                    def response = sh(
                        script: '''
                            set -e

                            curl -sS --fail \
                              -H "Accept: application/vnd.github+json" \
                              -H "Authorization: Bearer $GITHUB_TOKEN" \
                              -H "X-GitHub-Api-Version: 2026-03-10" \
                              "https://api.github.com/repos/shivapaturi/catalogue/dependabot/alerts?per_page=100"
                        ''',
                        returnStdout: true
                    ).trim()

                    def json = readJSON text: response

                    def highCritical = json.findAll { alert ->
                        def severity = alert?.security_advisory?.severity?.toLowerCase()
                        def state    = alert?.state?.toLowerCase()

                        state == 'open' &&
                        (severity == 'high' || severity == 'critical')
                    }

                    if (highCritical.size() > 0) {
                        echo "❌ Found ${highCritical.size()} HIGH/CRITICAL Dependabot alerts"
                        error 'Build failed because of HIGH/CRITICAL Dependabot vulnerabilities'
                    }

                    echo '✅ No HIGH/CRITICAL Dependabot alerts found'
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withAWS(
                        credentials: 'aws-creds',
                        region: "${env.REGION}"
                    ) {
                        sh """
                            set -e

                            ECR_REPO="${env.ACC_ID}.dkr.ecr.${env.REGION}.amazonaws.com/${env.PROJECT}/${env.COMPONENT}"

                            echo "Logging into ECR..."
                            aws ecr get-login-password --region "${env.REGION}" | \
                              docker login \
                              --username AWS \
                              --password-stdin "${env.ACC_ID}.dkr.ecr.${env.REGION}.amazonaws.com"

                            echo "Building image:"
                            echo "\$ECR_REPO:${env.APP_VERSION}"

                            docker buildx build \
                              --platform linux/amd64 \
                              --provenance=false \
                              --sbom=false \
                              --load \
                              -t "\$ECR_REPO:${env.APP_VERSION}" .

                            echo "Pushing image..."
                            docker push "\$ECR_REPO:${env.APP_VERSION}"
                        """
                    }
                }
            }
        }

        stage('Verify ECR Image') {
            steps {
                script {
                    withAWS(
                        credentials: 'aws-creds',
                        region: "${env.REGION}"
                    ) {
                        sh """
                            set -e

                            aws ecr describe-images \
                              --repository-name "${env.PROJECT}/${env.COMPONENT}" \
                              --image-ids imageTag="${env.APP_VERSION}" \
                              --region "${env.REGION}" \
                              --query 'imageDetails[0].{
                                  Tag:imageTags[0],
                                  Digest:imageDigest,
                                  MediaType:imageManifestMediaType
                              }' \
                              --output table
                        """
                    }
                }
            }
        }

        stage('Check Scan Results') {
            steps {
                script {
                    withAWS(
                        credentials: 'aws-creds',
                        region: "${env.REGION}"
                    ) {

                        def repository = "${env.PROJECT}/${env.COMPONENT}"
                        def imageTag   = "${env.APP_VERSION}"
                        def region     = "${env.REGION}"

                        echo "========================================"
                        echo "ECR Security Scan"
                        echo "Repository : ${repository}"
                        echo "Image Tag  : ${imageTag}"
                        echo "========================================"

                        // Confirm image exists
                        sh """
                            aws ecr describe-images \
                              --repository-name "${repository}" \
                              --image-ids imageTag="${imageTag}" \
                              --region "${region}"
                        """

                        echo "Waiting for ECR scan..."

                        retry(12) {

                            def result = sh(
                                script: """
                                    aws ecr describe-image-scan-findings \
                                      --repository-name "${repository}" \
                                      --image-id imageTag="${imageTag}" \
                                      --region "${region}" \
                                      --output json
                                """,
                                returnStatus: true
                            )

                            if (result != 0) {
                                echo "ECR scan results not available yet"
                                sleep time: 15, unit: 'SECONDS'
                                error "Retry ECR scan"
                            }

                            def findings = sh(
                                script: """
                                    aws ecr describe-image-scan-findings \
                                      --repository-name "${repository}" \
                                      --image-id imageTag="${imageTag}" \
                                      --region "${region}" \
                                      --output json
                                """,
                                returnStdout: true
                            ).trim()

                            def json = readJSON text: findings

                            def status =
                                json.imageScanStatus?.status?.toString()

                            echo "ECR Scan Status: ${status}"

                            if (status == 'IN_PROGRESS' ||
                                status == 'PENDING') {

                                echo 'Scan still running...'

                                sleep time: 15, unit: 'SECONDS'
                                error 'Scan not completed yet'
                            }

                            if (status != 'COMPLETE') {
                                error(
                                    "❌ ECR scan did not complete. Status: ${status}"
                                )
                            }

                            def counts =
                                json.imageScanFindings?.findingSeverityCounts ?: [:]

                            int low =
                                (counts.LOW ?: 0) as int

                            int medium =
                                (counts.MEDIUM ?: 0) as int

                            int high =
                                (counts.HIGH ?: 0) as int

                            int critical =
                                (counts.CRITICAL ?: 0) as int

                            echo "========================================"
                            echo "ECR Vulnerability Summary"
                            echo "LOW      : ${low}"
                            echo "MEDIUM   : ${medium}"
                            echo "HIGH     : ${high}"
                            echo "CRITICAL : ${critical}"
                            echo "========================================"

                            if (high > 0 || critical > 0) {

                                error(
                                    "❌ Security gate FAILED: " +
                                    "${high} HIGH, ${critical} CRITICAL vulnerabilities found"
                                )
                            }

                            echo '✅ Security gate PASSED'
                            echo '✅ No HIGH/CRITICAL vulnerabilities found'
                        }
                    }
                }
            }
        }

        stage('Trigger Deploy') {
            when {
                expression {
                    params.deploy
                }
            }

            steps {
                script {
                    echo "Triggering catalogue-cd with version ${env.APP_VERSION}"

                    build job: 'catalogue-cd',
                        parameters: [
                            string(
                                name: 'appVersion',
                                value: "${env.APP_VERSION}"
                            ),
                            string(
                                name: 'deploy_to',
                                value: 'dev'
                            )
                        ],
                        propagate: false,
                        wait: false
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