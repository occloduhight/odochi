pipeline {
    agent any
    environment {
        NEXUS_USER = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO = credentials('nexus-repo')
    }
    stages {
        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    // Use the full plugin coordinates to avoid NoPluginFoundForPrefixException
                    sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:3.10.0.1446:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build Artifact') {
            steps {
                sh 'mvn clean package -DskipTests -Dcheckstyle.skip'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/petclinicapps .'
            }
        }

        stage('Push Artifact to Nexus Repo') {
            steps {
                nexusArtifactUploader(
                    artifacts: [[
                        artifactId: 'spring-petclinic',
                        classifier: '',
                        file: 'target/spring-petclinic-2.4.2.war',
                        type: 'war'
                    ]],
                    credentialsId: 'nexus-creds',
                    groupId: 'Petclinic',
                    nexusUrl: 'nexus.tundeafod.click',
                    nexusVersion: 'nexus3',
                    protocol: 'https',
                    repository: 'nexus-repo',
                    version: '1.0'
                )
            }
        }

        stage('Trivy fs Scan') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage('Docker Login & Push') {
            steps {
                sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO'
                sh 'docker push $NEXUS_REPO/petclinicapps'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image $NEXUS_REPO/petclinicapps > trivyimage.txt"
            }
        }

        stage('Deploy to Stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh 'ssh -t -t ec2-user@3.8.33.146 -o StrictHostKeyChecking=no "ansible-playbook -i /etc/ansible/stage-hosts /etc/ansible/stage-playbook.yml"'
                }
            }
        }

        stage('Check Stage Website') {
            steps {
                sleep 90
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://stage.tundeafod.click", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "Stage app is up: HTTP ${response}", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "Stage app down: HTTP ${response}", tokenCredentialId: 'slack')
                    }
                }
            }
        }

        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval', submitter: 'admin'
                }
            }
        }

        stage('Deploy to Prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh 'ssh -t -t ec2-user@3.8.33.146 -o StrictHostKeyChecking=no "ansible-playbook -i /etc/ansible/prod-hosts /etc/ansible/prod-playbook.yml"'
                }
            }
        }

        stage('Check Prod Website') {
            steps {
                sleep 90
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://prod.tundeafod.click", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "Prod app is up: HTTP ${response}", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "Prod app down: HTTP ${response}", tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}
