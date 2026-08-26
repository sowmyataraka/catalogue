pipeline {
    agent { 
        node { 
            label 'roboshop' 
        } 
    }
    environment {
        def appVersion = ""
        acc_id = "841747068918"
        project = "roboshop"
        component = "catalogue"
    }
    options {
        disableConcurrentBuilds()
        timeout(time: 15, unit: 'MINUTES')
    }
    /* parameters {
        string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
        text(name: 'BIOGRAPHY', defaultValue: '', description: 'Enter some information about the person')
        booleanParam(name: 'DEPLOY', defaultValue: true, description: 'Toggle this value')
        choice(name: 'CHOICE', choices: ['One', 'Two', 'Three'], description: 'Pick something')
        password(name: 'PASSWORD', defaultValue: 'SECRET', description: 'Enter a password')
    } */
    // Build
    stages {
        stage('Read version'){
            steps{
                script {
                    def packageJson = readJSON file: 'package.json'
                    // Extract the version property
                    appVersion = packageJson.version
                    echo "The application version is: ${appVersion}"
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
        // this command gives us coverage report and test cases report, sonarqube access this to check quality gate
        stage('Unit tests') {
            steps {
                script {
                    sh """
                        npm test
                    """
                } 
            }
        }
        /* stage('SonarQube Analysis') {
            steps {
                // 'My SonarQube Server' must match the name configured in Jenkins System Settings
                withSonarQubeEnv('sonar-server') {
                    sh "${tool 'sonar-8'}/bin/sonar-scanner"
                }
            }
        }
        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate() // Pauses pipeline
                        if (qg.status != 'OK') {
                            error "Pipeline aborted: ${qg.status}"
                        }
                    }
                }
            }
        } */
        stage('Check Dependency Vulnerabilities') {
    steps {
        withCredentials([string(
            credentialsId: 'github-pat',
            variable: 'GITHUB_TOKEN'
        )]) {

            script {
                def response = sh(
                    script: '''
                        curl -sS -L \
                          -H "Accept: application/vnd.github+json" \
                          -H "Authorization: Bearer ${GITHUB_TOKEN}" \
                          -H "X-GitHub-Api-Version: 2026-03-10" \
                          "https://api.github.com/repos/sowmyataraka/catalogue/dependabot/alerts"
                    ''',
                    returnStdout: true
                ).trim()

                def alerts = readJSON text: response

                def dangerousAlerts = alerts.findAll { alert ->
                    alert.state == 'open' &&
                    ['high', 'critical'].contains(
                        alert.security_vulnerability.severity?.toLowerCase()
                    )
                }

                if (dangerousAlerts) {
                    echo "❌ High/Critical dependency vulnerabilities found:"

                    dangerousAlerts.each { alert ->
                        echo """
Alert #${alert.number}
Package: ${alert.dependency.package.name}
Severity: ${alert.security_vulnerability.severity}
CVE: ${alert.security_advisory.cve_id}
Summary: ${alert.security_advisory.summary}
Fixed version: ${alert.security_vulnerability.first_patched_version?.identifier}
"""
                    }

                    error("Build failed: High/Critical dependency vulnerabilities detected.")
                }

                echo "✅ No open High/Critical dependency vulnerabilities found."
            }
        }
    }
}
        stage('Docker Build') {
            steps {
                script {
                    // in this block we get aws authentication
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${acc_id}.dkr.ecr.us-east-1.amazonaws.com
                            docker build -t ${acc_id}.dkr.ecr.us-east-1.amazonaws.com/${project}/${component}:${appVersion} .
                        """
                    }
                }
            }
        }
        stage('Trivy Scan') {
            steps {
                script {
                    def dockerfileScan = sh(
                        script: """
                            trivy config --exit-code 1 --severity HIGH,CRITICAL --format table ./Dockerfile
                        """,
                        returnStatus: true
                    )

                    def imageScan = sh(
                        script: """
                            trivy image --scanners vuln --pkg-types os --exit-code 1 --severity HIGH,CRITICAL --format table ${acc_id}.dkr.ecr.us-east-1.amazonaws.com/${project}/${component}:${appVersion}
                        """,
                        returnStatus: true
                    )

                    if (dockerfileScan != 0 || imageScan != 0) {
                        error "Trivy found HIGH/CRITICAL issues in Dockerfile and/or OS packages. Failing pipeline."
                    }
                }
            }
        }
        stage('ECR Image push') {
            steps {
                script {
                    // in this block we get aws authentication
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${acc_id}.dkr.ecr.us-east-1.amazonaws.com
                            docker push ${acc_id}.dkr.ecr.us-east-1.amazonaws.com/${project}/${component}:${appVersion}
                        """
                    }
                }
            }
        }
        stage('Deploy') {
            when {
                // Evaluates the boolean parameter directly
                expression { "${params.DEPLOY}" == "true" }
            }
            /* input {
                message "Should we continue?"
                ok "Yes, we should."
                submitter "alice,bob"
                parameters {
                    string(name: 'PERSON', defaultValue: 'Mr Jenkins', description: 'Who should I say hello to?')
                }
            } */
            steps {
                script {
                    sh """
                        echo "Deploying"
                    """
                }
            }
        }
    }

    post { 
        always { 
            echo 'I will always say Hello again!'
        }
        success { 
            echo 'I will run when success'
        }
        failure { 
            echo 'I will Run when it is failed'
        }
    }
}