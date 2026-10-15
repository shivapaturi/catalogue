@Library('jenkins-shared-library@main') _

def configMap = [
    project : "roboshop",
    component : "catalogue"
]
if ( !env.BRANCH_NAME.equalsIgnoreCase('main') ){ // This means any branch except main.
    nodejsEKSPipeline(configMap) // by  default it will call, call fuction inside this pipeline 
}
else{
    echo "Please proceed  with PROD process"
}
