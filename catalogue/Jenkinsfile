pipeline{
    agent {
        node{
            label 'roboshop'
        }
    }
    environment {
        appVersion = ""
        ACC_ID = "180970910849"
        REGION = "us-east-1"
    }

    options {
        //disableConcurrentBuilds()
        timeout(time: 5, unit: 'MINUTES')
    }

    /* parameters {
        string(name: 'PERSON', defaultValue: 'vedavyas', description: 'Who to greet?')
        choice(name: 'ENV', choices: ['Dev', 'Staging', 'Prod'], description: 'Target environment')
        booleanParam(name: 'DEBUG', defaultValue: true, description: 'Enable debug logs')
    } */

    stages{
        stage ('Read Version') {
            steps {
                script {
                    // Read the file from the workspace
                    def packageJson = readJSON file: 'package.json'
                    
                    // Access properties directly
                    appVersion = packageJson.version
                    echo "Building version ${appVersion}"
                }
            }        
        }

        stage ('Install Dependencies'){
            steps{
                script {
                    sh """ npm install """
                }          
            }
        }

        stage ('Build Image'){
            steps{
                script {
                    withAWS(credentials: 'aws-creds', region: ${REGION}) {                        
                    // You can also run standard CLI commands inside this block
                    sh """
                        aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com
                        docker build -t ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/roboshop/catalogue:${appVersion} .
                        docker push ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/roboshop/catalogue:${appVersion}
                    """
                    }
                }          
            }
        }
        
    }

    post {
        always {
            echo "Hi vedavyas"
        }
        success {
            echo "pipeline success"
        }
        failure {
            echo "pipeline failure"
        }
        
    }
}