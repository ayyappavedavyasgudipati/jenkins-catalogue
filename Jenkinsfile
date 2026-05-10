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

        stage ('Unit Test Cases'){
            steps{
                script {
                    sh """ npm test """
                }          
            }
        }

        stage('Dependabot Alerts Check') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    script {
                        def owner = 'ayyappavedavyasgudipati'
                        def repo  = 'jenkins-catalogue'

                        def response = sh(
                            script: """
                                curl -s -w "\\n%{http_code}" \\
                                    -H "Authorization: Bearer ${GITHUB_TOKEN}" \\
                                    -H "Accept: application/vnd.github+json" \\
                                    -H "X-GitHub-Api-Version: 2022-11-28" \\
                                    "https://api.github.com/repos/${owner}/${repo}/dependabot/alerts?severity=high,critical&state=open&per_page=100"
                            """,
                            returnStdout: true
                        ).trim()

                        def parts      = response.tokenize('\n')
                        def httpStatus = parts[-1].trim()
                        def body       = parts[0..-2].join('\n')

                        if (httpStatus != '200') {
                            error "GitHub API call failed with HTTP ${httpStatus}. Check token permissions (security_events scope required).\nResponse: ${body}"
                        }

                        def alerts = readJSON text: body

                        if (alerts.size() == 0) {
                            echo "✅ No HIGH or CRITICAL Dependabot alerts found. Pipeline continues."
                        } else {
                            echo "🚨 Found ${alerts.size()} HIGH/CRITICAL Dependabot alert(s):"
                            alerts.each { alert ->
                                def pkg      = alert.security_vulnerability?.package?.name ?: 'unknown'
                                def severity = alert.security_advisory?.severity?.toUpperCase() ?: 'UNKNOWN'
                                def summary  = alert.security_advisory?.summary ?: 'No summary'
                                def fixedIn  = alert.security_vulnerability?.first_patched_version?.identifier ?: 'No fix available'
                                echo "  ❌ [${severity}] ${pkg} — ${summary} (Fixed in: ${fixedIn})"
                            }
                            error "Pipeline failed: ${alerts.size()} HIGH/CRITICAL Dependabot alert(s) detected."
                        }
                    }
                }
            }
        }

        stage ('Build Image'){
            steps{
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {                        
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

        tage('Trivy OS Scan') {
            steps {
                script {
                    // Generate table report
                    sh """
                        trivy image \
                            --scanners vuln \
                            --pkg-types os \
                            --severity HIGH,MEDIUM \
                            --format table \
                            --output trivy-os-report.txt \
                            --exit-code 0 \
                            ${ACC_ID}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${appVersion}
                    """

                    // Print table to console
                    sh 'cat trivy-os-report.txt'

                    // Fail pipeline if vulnerabilities found
                    def scanResult = sh(
                        script: """
                            trivy image \
                                --scanners vuln \
                                --pkg-types os \
                                --severity HIGH,MEDIUM \
                                --format table \
                                --exit-code 1 \
                                --quiet \
                                ${ACC_ID}.dkr.ecr.${region}.amazonaws.com/roboshop/catalogue:${appVersion}
                        """,
                        returnStatus: true
                    )

                    if (scanResult != 0) {
                        error "🚨 Trivy found HIGH/MEDIUM OS vulnerabilities. Pipeline failed."
                    } else {
                        echo "✅ No HIGH or MEDIUM OS vulnerabilities found. Pipeline continues."
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