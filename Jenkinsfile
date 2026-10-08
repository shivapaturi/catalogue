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
                    def packageJson = readJSON file: 'package.json'
                    appVersion = packageJson.version
                    echo "package version: ${appVersion}"
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