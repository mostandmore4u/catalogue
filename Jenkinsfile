pipeline {
    agent { 
        node { 
            label 'ROBOSHOP' 
        } 
    }
    environment {
        acc_id = "722741356463"
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
                    env.appVersion = packageJson.version
                    echo "The application version is: ${env.appVersion}"
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
        stage('Unit tests') {
            steps {
                script{
                    sh """
                        npm test
                    """
                }
            }
        }
  /*       stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-8'
                    echo "Scanner home: ${scannerHome}"
                    withSonarQubeEnv('sonar-server') {
                        sh "${scannerHome}/bin/sonar-scanner"
                    }
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
        stage('Dependabot Alerts') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        set +x
                        HTTP_CODE=$(curl -sS -L -o dependabot-alerts.json -w "%{http_code}" -H "Accept: application/vnd.github+json" -H "Authorization: Bearer $GITHUB_TOKEN" -H "X-GitHub-Api-Version: 2026-03-10" "https://api.github.com/repos/mostandmore4u/catalogue/dependabot/alerts?state=open&severity=high,critical&per_page=100")
                        if [ "$HTTP_CODE" != "200" ]; then
                            echo "GitHub API returned HTTP $HTTP_CODE - cannot verify Dependabot alerts"
                            cat dependabot-alerts.json
                            exit 2
                        fi
                        node -e '
                            const alerts = require("./dependabot-alerts.json");
                            if (!Array.isArray(alerts)) { console.error("Unexpected API response"); process.exit(2); }
                            const bad = alerts.filter(a => ["high", "critical"].includes(a.security_advisory.severity));
                            if (bad.length === 0) { console.log("No open high/critical Dependabot alerts"); process.exit(0); }
                            console.error(`Found ${bad.length} open high/critical Dependabot alert(s):`);
                            bad.forEach(a => {
                                const fix = a.security_vulnerability.first_patched_version ? a.security_vulnerability.first_patched_version.identifier : "none";
                                console.error(` - #${a.number} [${a.security_advisory.severity}] ${a.dependency.package.name} ${a.security_advisory.cve_id || a.security_advisory.ghsa_id} (fixed in ${fix}) ${a.html_url}`);
                            });
                            process.exit(1);
                        '
                    '''
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
                            docker build -t ${acc_id}.dkr.ecr.us-east-1.amazonaws.com/${project}/${component}:${env.appVersion} .
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
                            trivy image --scanners vuln --pkg-types os --exit-code 1 --severity HIGH,CRITICAL --format table ${IMAGE_NAME}:${IMAGE_TAG}
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