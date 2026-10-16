@Library('jenkins-shared-library@main') _

def configMap = [
    project   : "roboshop",
    component : "catalogue"
]

def branchName = env.BRANCH_NAME ?: ''

echo "BRANCH_NAME = ${branchName}"

if (!branchName.equalsIgnoreCase('main')) {
    nodejsEKSPipeline(configMap)
} else {
    echo "Please proceed with PROD process"
}