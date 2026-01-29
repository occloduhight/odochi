pipeline {
    agent any
    environment {
        NEXUS_CRED = credentials('nexus-cred')          // Nexus username/password
        NEXUS_REPO = credentials('nexus-docker-repo')  // Nexus Docker registry (host only)
        NVD_API_KEY = credentials('nvd-key')           // Trivy NVD API key
        BASTION_IP = credentials('bastion-ip')         // Bastion host IP
        ANSIBLE_IP = credentials('ansible-ip')         // Ansible server IP
        SONAR_TOKEN = credentials('sonarqube')         // SonarQube token
        SLACK_TOKEN = credentials('slack')             // Slack bot token
    }

    stage('Code analysis stage') {
    steps {
        sh '''
            export SONAR_TOKEN=${SONAR_TOKEN}
            mvn sonar:sonar \
              -Dsonar.host.url=https://sonar.odochidevops.space/ \
              -Dsonar.login=${SONAR_TOKEN}
        '''
    }
}


        stage('Quality gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build artefacts') {
            steps {
                sh 'mvn clean package -DskipTests -Dcheckstyle.skip'
            }
        }

        stage('Push artifacts to Nexus Maven repo') {
            steps {
                nexusArtifactUploader(
                    artifacts: [[
                        artifactId: 'spring-petclinic',
                        classifier: '',
                        file: 'target/spring-petclinic-2.4.2.war',
                        type: 'war'
                    ]],
                    credentialsId: 'nexus-cred',
                    groupId: 'Petclinic',
                    nexusUrl: 'nexus.odochidevops.space',
                    nexusVersion: 'nexus3',
                    protocol: 'https',
                    repository: 'nexus-repo',
                    version: '1.0'
                )
            }
        }

        stage('Build Docker image') {
            steps {
                sh 'docker build -t $NEXUS_REPO/petclinicapps:latest .'
            }
        }

        stage('Login to Nexus Docker repo') {
            steps {
                sh 'docker login -u $NEXUS_CRED_USR -p $NEXUS_CRED_PSW $NEXUS_REPO'
            }
        }

        stage('Trivy image scan') {
            steps {
                sh "trivy image -f table $NEXUS_REPO/petclinicapps:latest > trivy.txt"
            }
        }

        stage('Push Docker image to Nexus') {
            steps {
                sh 'docker push $NEXUS_REPO/petclinicapps:latest'
            }
        }

        stage('Prune Docker image from Jenkins server') {
            steps {
                sh 'docker rmi $NEXUS_REPO/petclinicapps:latest'
            }
        }

        stage('Deploy to stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                        ssh -t -t -o StrictHostKeyChecking=no \
                        -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ec2-user@${BASTION_IP}" \
                        ec2-user@${ANSIBLE_IP} "ansible-playbook -i /etc/ansible/stage_hosts /etc/ansible/deployment.yml"
                    '''
                }
            }
        }

        stage('Check stage website availability') {
            steps {
                sh 'sleep 90'
                script {
                    def response = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://stage.wahd.solutions', returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "Stage petclinic website is up (HTTP ${response})", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "Stage petclinic website appears down (HTTP ${response})", tokenCredentialId: 'slack')
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

        stage('Deploy to prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh '''
                        ssh -t -t -o StrictHostKeyChecking=no \
                        -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ec2-user@${BASTION_IP}" \
                        ec2-user@${ANSIBLE_IP} "ansible-playbook -i /etc/ansible/prod_hosts /etc/ansible/deployment.yml"
                    '''
                }
            }
        }

        stage('Check prod website availability') {
            steps {
                sh 'sleep 90'
                script {
                    def response = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://prod.wahd.solutions', returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "Prod petclinic website is up (HTTP ${response})", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "Prod petclinic website appears down (HTTP ${response})", tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}
