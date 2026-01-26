pipeline {
    agent any

    environment {
        NEXUS_REPO = 'nexus.tundeafod.click/repository/nexus-repo'
        NEXUS_HOST = 'nexus.tundeafod.click'
        NEXUS_IP   = '10.0.1.5'
        IMAGE_NAME = 'spring-petclinic:2.4.2'
    }

    stages {

        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn clean verify sonar:sonar -Dcheckstyle.skip'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Dependency Check') {
            steps {
                dependencyCheck odcInstallation: 'DP-Check'
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
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Login & Push') {
            steps {
                script {

                    def resolved = sh(
                        script: "getent hosts ${NEXUS_HOST} || echo NOTFOUND",
                        returnStdout: true
                    ).trim()

                    if (resolved == 'NOTFOUND') {
                        echo "DNS failed. Mapping ${NEXUS_HOST} → ${NEXUS_IP}"
                        sh "echo '${NEXUS_IP} ${NEXUS_HOST}' | sudo tee -a /etc/hosts"
                    } else {
                        echo "${NEXUS_HOST} already resolvable"
                    }

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'nexus-repo',
                            usernameVariable: 'USER',
                            passwordVariable: 'PASS'
                        )
                    ]) {
                        sh """
                            echo \$PASS | docker login -u \$USER --password-stdin ${NEXUS_REPO}
                            docker tag ${IMAGE_NAME} ${NEXUS_REPO}/${IMAGE_NAME}
                            docker push ${NEXUS_REPO}/${IMAGE_NAME}
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${NEXUS_REPO}/${IMAGE_NAME} > trivyimage.txt"
            }
        }

        stage('Deploy to Stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@3.8.33.146 \
                        "ansible-playbook -i /etc/ansible/stage-hosts /etc/ansible/stage-playbook.yml"
                    '''
                }
            }
        }

        stage('Check Stage Website') {
            steps {
                retry(3) {
                    sleep 30
                    script {
                        def status = sh(
                            script: 'curl -s -o /dev/null -w "%{http_code}" https://stage.tundeafod.click',
                            returnStdout: true
                        ).trim()

                        if (status != '200') {
                            error "Stage not ready (HTTP ${status})"
                        }

                        slackSend(
                            color: 'good',
                            message: "Stage app is live (HTTP ${status})",
                            tokenCredentialId: 'slack'
                        )
                    }
                }
            }
        }

        stage('Request Approval for Prod') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Approve deployment to PROD?', submitter: 'admin'
                }
            }
        }

        stage('Deploy to Prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ec2-user@3.8.33.146 \
                        "ansible-playbook -i /etc/ansible/prod-hosts /etc/ansible/prod-playbook.yml"
                    '''
                }
            }
        }

        stage('Check Prod Website') {
            steps {
                retry(3) {
                    sleep 30
                    script {
                        def status = sh(
                            script: 'curl -s -o /dev/null -w "%{http_code}" https://prod.tundeafod.click',
                            returnStdout: true
                        ).trim()

                        if (status != '200') {
                            error "Prod not ready (HTTP ${status})"
                        }

                        slackSend(
                            color: 'good',
                            message: "Prod app is live (HTTP ${status})",
                            tokenCredentialId: 'slack'
                        )
                    }
                }
            }
        }
    }
}
