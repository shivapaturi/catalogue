pipeline {
    agent {
        node {
            label 'AGENT-1'
        }
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
    stages {
        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    env.appVersion = packageJson.version
                    echo "package version: ${env.appVersion}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('Docker Build') {
            steps {
                withAWS(credentials: 'aws-creds', region: "${REGION}") {
                    sh """
                        aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com
                        docker build -t ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${env.appVersion} .
                        docker push ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}/${COMPONENT}:${env.appVersion}
                    """
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