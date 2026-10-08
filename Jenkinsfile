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
                // Timeout counter starts BEFORE agent is allocated
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        }
    }    
    // parameters {
    //     string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
    //     text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')
    //     booleanParam(name: 'TOGGLE', defaultValue: true, description: 'Toggle this value')
    //     choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')
    //     password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
    // }
    stages {
        stage('Read package.json') {
            steps {
                script {
                    def pkg = readJSON file: 'package.json'
                    appVersion = package.json.version
                    echo "package version: ${appVersion}"
                }
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
