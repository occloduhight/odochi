pipeline {
    agent any

    environment {
        NEXUS_REPO = 'nexus.tundeafod.click/repository/maven-releases'
    }

    stage('Code Analysis') {
    steps {
        withSonarQubeEnv('sonar') {
            sh 'mvn clean verify sonar:sonar -Dcheckstyle.skip'
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
        dependencyCheck additionalArguments: '--disableExtensions "NoHttp"', odcInstallation: 'DP-Check'

                        odcInstallation: 'DP-Check'
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
                sh 'docker build -t petclinicapps .'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        // ===================== Remaining Stages =====================

        stage('Docker Login & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-username-password', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh 'docker login -u $USER -p $PASS $NEXUS_REPO'
                    sh 'docker push $NEXUS_REPO/petclinicapps'
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image $NEXUS_REPO/petclinicapps > trivyimage.txt'
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
                retry(3) {
                    sleep 30
                    script {
                        def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://stage.tundeafod.click", returnStdout: true).trim()
                        if (response != "200") {
                            error("Stage site not ready yet: HTTP ${response}")
                        } else {
                            slackSend(color: 'good', message: "Stage app is up: HTTP ${response}", tokenCredentialId: 'slack')
                        }
                    }
                }
            }
        }

        stage('Request Approval for Prod') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Approve deployment to PROD?', submitter: 'admin'
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
                retry(3) {
                    sleep 30
                    script {
                        def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://prod.tundeafod.click", returnStdout: true).trim()
                        if (response != "200") {
                            error("Prod site not ready yet: HTTP ${response}")
                        } else {
                            slackSend(color: 'good', message: "Prod app is up: HTTP ${response}", tokenCredentialId: 'slack')
                        }
                    }
                }
            }
        }

    } // end of stages
}
