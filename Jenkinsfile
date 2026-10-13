pipeline {
    agent {
        node {
            label 'AGENT-1'
        }
    }

    environment {
        REGION    = "us-east-1"
        ACC_ID    = "211125615103"
        PROJECT   = "roboshop"
        COMPONENT = "catalogue"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    parameters {
        booleanParam(name: 'deploy', defaultValue: false, description: 'Toggle this value')
    }
    stages {

        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    env.appVersion = packageJson.version

                    echo "Package version: ${env.appVersion}"
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
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
        // stage('Sonar Scan') {
        //     environment {
        //         scannerHome = tool 'sonar-8.1'
        //     }
        //     steps {
        //         script {
        //            // Sonar Server envrionment
        //            withSonarQubeEnv(installationName: 'sonar-8.1') {
        //                  sh "${scannerHome}/bin/sonar-scanner"
        //            }
        //         }
        //     }
        // }
        // Enable webhook in sonarqube server And wait for Results  
        // stage("Quality Gate") {
        //     steps {
        //         timeout(time: 1, unit: 'HOURS') {
        //         waitForQualityGate abortPipeline: true
        //         }
        //     }
        // }
        stage('Check Dependabot Alerts') {
            environment {
                GITHUB_TOKEN = credentials('github-token')
            }

            steps {
                script {

                    def response = sh(
                        script: '''
                            set -e

                            curl -sS -L \
                            -H "Accept: application/vnd.github+json" \
                            -H "Authorization: Bearer $GITHUB_TOKEN" \
                            -H "X-GitHub-Api-Version: 2026-03-10" \
                            "https://api.github.com/repos/shivapaturi/catalogue/dependabot/alerts?state=open&severity=high,critical&per_page=100"
                        ''',
                        returnStdout: true
                    ).trim()

                    def json = readJSON(
                        text: response,
                        returnPojo: true
                    )

                    if (!(json instanceof List)) {
                        error "❌ GitHub API did not return a Dependabot alert list: ${json}"
                    }

                    if (json.size() > 0) {

                        echo "❌ Found ${json.size()} OPEN HIGH/CRITICAL Dependabot alerts."

                        json.each { alert ->

                            def packageName =
                                alert?.dependency?.package?.name ?: 'Unknown'

                            def ecosystem =
                                alert?.dependency?.package?.ecosystem ?: 'Unknown'

                            def severity =
                                alert?.security_advisory?.severity ?: 'Unknown'

                            def cve =
                                alert?.security_advisory?.cve_id ?: 'N/A'

                            def vulnerableRange =
                                alert?.security_vulnerability?.vulnerable_version_range ?: 'N/A'

                            def fixedVersion =
                                alert?.security_vulnerability?.first_patched_version?.identifier ?: 'N/A'

                            echo """
                            ----------------------------------------
                            Package          : ${packageName}
                            Ecosystem        : ${ecosystem}
                            Severity          : ${severity}
                            CVE               : ${cve}
                            Vulnerable range  : ${vulnerableRange}
                            First fixed       : ${fixedVersion}
                            ----------------------------------------
                            """
                        }

                        error "❌ Security gate failed: HIGH/CRITICAL Dependabot vulnerabilities found."
                    }

                    echo "✅ No OPEN HIGH/CRITICAL Dependabot alerts found."
                }
            }

        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withAWS(
                        credentials: 'aws-creds',
                        region: "${REGION}"
                    ) {
                        sh """
                            echo "Logging into AWS ECR..."

                            aws ecr get-login-password --region ${REGION} | \
                            docker login \
                            --username AWS \
                            --password-stdin ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com

                            echo "Building Docker image..."

                            docker build \
                            -t ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion} .

                            echo "Pushing Docker image..."

                            docker push \
                            ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion}
                        """
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
            echo 'Hello success!'
        }

        failure {
            echo 'Hello failure!'
        }
    }
}