pipeline {
    agent {
        node {
            label 'AGENT-1'
        }
    }
    environment {
        appVersion = ''
    }
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    stages {
        stage('Read package.json') {
            steps {
                script {
                    def pkg = readJSON file: 'package.json'
                    env.appVersion = pkg.version
                    echo "package version: ${env.appVersion}"
                }
            }
        }
        stage('Test') {
            steps {
                script {
                    echo 'Testing..'
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    echo 'Deploying....'
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