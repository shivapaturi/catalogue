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

    stages {

        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    env.APP_VERSION = packageJson.version

                    echo "Package version: ${env.APP_VERSION}"
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
                            -t ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${APP_VERSION} .

                            echo "Pushing Docker image..."

                            docker push \
                            ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${APP_VERSION}
                        """
                    }
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