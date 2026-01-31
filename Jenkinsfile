pipeline {
    agent any

    environment {
        NEXUS_URL      = 'nexus.odochidevops.space'
        NEXUS_USER     = credentials('nexus-docker-username')
        NEXUS_PASSWORD= credentials('nexus-docker-password')
        DOCKER_IMAGE   = 'nexus.odochidevops.space/docker-hosted/apppetclinic:2.4.2'
        ANSIBLE_IP     = credentials('ansible-ip')
        BASTION_ID     = credentials('bastion-id')
        NVD_API_KEY    = credentials('nvd-key')
        AWS_REGION     = 'eu-west-3'
    }

    triggers {
        pollSCM('* * * * *') // Runs every minute
    }

    stages {

        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
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
                dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey ${NVD_API_KEY}",
                                odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build Artifact') {
            steps {
                sh 'mvn clean package -DskipTests -Dcheckstyle.skip'
            }
        }

        stage('Push Artifact to Nexus Repo') {
            steps {
                nexusArtifactUploader artifacts: [[
                    artifactId: 'spring-petclinic',
                    classifier: '',
                    file: 'target/spring-petclinic-2.4.2.war',
                    type: 'war'
                ]],
                credentialsId: 'nexus-maven-cred',
                groupId: 'Petclinic',
                nexusUrl: 'nexus.odochidevops.space',
                nexusVersion: 'nexus3',
                protocol: 'https',
                repository: 'nexus-maven-repo',
                version: '1.0'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Docker: Login and Push to Nexus') {
            steps {
                sh '''
                  echo "$NEXUS_PASSWORD" | docker login nexus.odochidevops.space \
                    --username "$NEXUS_USER" \
                    --password-stdin
                '''
                sh "docker push ${DOCKER_IMAGE}"
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image -f table ${DOCKER_IMAGE} > trivy-report.txt"
            }
        }

        stage('Prune Docker Images') {
            steps {
                sh 'docker image prune -f'
            }
        }

        stage('Deploy to Stage') {
            steps {
                script {
                    sh '''
                      aws ssm start-session \
                        --target ${BASTION_ID} \
                        --region ${AWS_REGION} \
                        --document-name AWS-StartPortForwardingSession \
                        --parameters '{"portNumber":["22"],"localPortNumber":["9998"]}' &
                      sleep 5
                    '''

                    sshagent(['bastion-key', 'ansible-key']) {
                        sh '''
                          ssh -o StrictHostKeyChecking=no \
                              -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9998" \
                              ec2-user@${ANSIBLE_IP} \
                              "ansible-playbook -i /etc/ansible/stage_hosts /etc/ansible/deployment.yml"
                        '''
                    }

                    sh 'pkill -f "aws ssm start-session"'
                }
            }
        }

        stage('Check Stage Availability') {
            steps {
                sh 'sleep 90'
                script {
                    def code = sh(
                        script: 'curl -s -o /dev/null -w "%{http_code}" https://stage.odochidevops.space',
                        returnStdout: true
                    ).trim()

                    if (code == '200') {
                        slackSend(color: 'good',
                                  message: "Stage environment is UP (HTTP ${code})",
                                  tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger',
                                  message: "Stage environment DOWN (HTTP ${code})",
                                  tokenCredentialId: 'slack')
                    }
                }
            }
        }

        stage('DAST Scan') {
            steps {
                sh '''
                  chmod 777 $(pwd)
                  docker run -v $(pwd):/zap/wrk/:rw \
                    -t ghcr.io/zaproxy/zaproxy:stable \
                    zap-baseline.py \
                    -t https://stage.odochidevops.space \
                    -g gen.conf \
                    -r testreport.html || true
                '''
            }
        }

        stage('Request for Approval') {
            steps {
                timeout(time: 10, activity: true) {
                    input message: 'Approve deployment to production?', submitter: 'admin'
                }
            }
        }

        stage('Deploy to Prod') {
            steps {
                script {
                    sh '''
                      aws ssm start-session \
                        --target ${BASTION_ID} \
                        --region ${AWS_REGION} \
                        --document-name AWS-StartPortForwardingSession \
                        --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' &
                      sleep 5
                    '''

                    sshagent(['bastion-key', 'ansible-key']) {
                        sh '''
                          ssh -o StrictHostKeyChecking=no \
                              -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                              ec2-user@${ANSIBLE_IP} \
                              "ansible-playbook -i /etc/ansible/prod_hosts /etc/ansible/deployment.yml"
                        '''
                    }

                    // sh 'pkill -f "aws ssm start-session"'
                }
            }
        }

        stage('Check Prod Availability') {
            steps {
                sh 'sleep 90'
                script {
                    def code = sh(
                        script: 'curl -s -o /dev/null -w "%{http_code}" https://prod.odochidevops.space',
                        returnStdout: true
                    ).trim()

                    if (code == '200') {
                        slackSend(color: 'good',
                                  message: "Production is UP (HTTP ${code})",
                                  tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger',
                                  message: "Production is DOWN (HTTP ${code})",
                                  tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}
