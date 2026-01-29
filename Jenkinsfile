pipeline{
    agent any
    environment {
        NEXUS_USER = credentials('nexus-docker-username')
        NEXUS_PASSWORD = credentials('nexus-docker-password')
        NEXUS_REPO = credentials('nexus-docker-url')
        ANSIBLE_IP = credentials('ansible-ip')
        NVD_API_KEY= credentials('nvd-key')
        BASTION_ID= credentials('bastion-id')
        AWS_REGION= 'eu-west-3'
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
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Dependency check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck additionalArguments: "--scan ./ --disableYarnAudit --disableNodeAudit --nvdApiKey $NVD_API_KEY",
                        odcInstallation: 'DP-Check'
                }
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
        script {
            try {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'https',
                    nexusUrl: 'nexus.odochidevops.space',
                    repository: 'nexus-maven-repo',
                    groupId: 'org.springframework.samples',
                    version: '2.4.2',
                    credentialsId: 'nexus-maven-cred',
                    artifacts: [
                        [
                            artifactId: 'spring-petclinic',
                            classifier: '',
                            file: 'target/spring-petclinic-2.4.2.war',
                            type: 'war'
                        ]
                    ]
                )
            } catch (e) {
                error "Nexus upload failed: ${e}"
            }
        }
    }
}


stage('Build Docker Image') {
    steps {
        sh '''
            docker build \
            -t ${NEXUS_REPO}/nexus-docker-repo/apppetclinic:2.4.2 \
            .
        '''
    }
}

        stage('Log Into Nexus Docker Repo') {
            steps {
                sh 'docker login --username $NEXUS_USER --password $NEXUS_PASSWORD $NEXUS_REPO'
            }
        }
        stage('Trivy image Scan') {
            steps {
                sh "trivy image -f table $NEXUS_REPO/nexus-docker-repo/apppetclinic > trivyfs.txt"
            }
        }
        stage('Push to Nexus Docker Repo') {
            steps {
                sh 'docker push $NEXUS_REPO/nexus-docker-repo/apppetclinic'
            }
        }
        stage('prune docker images') {
            steps {
                sh 'docker image prune -a -f'
            }
        }
       stage ('Deploying to Stage Environment') {
            steps {
               script {
                  // Start SSM session to bastion with port forwarding
                  sh '''
                    aws ssm start-session \
                      --target ${BASTION_ID} \
                      --region ${AWS_REGION} \
                      --document-name AWS-StartPortForwardingSession \
                      --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                      &
                    sleep 5
                  '''

                  // SSH into Bastion (via local port 9999), then hop to Ansible server
                  sshagent(['bastion-key', 'ansible-key']) {
                    sh '''
                      ssh -o StrictHostKeyChecking=no -p 9999 ubuntu@localhost \
                        "ssh -o StrictHostKeyChecking=no ec2-user@${ANSIBLE_IP} \
                          'ansible-playbook -i /etc/ansible/stage_hosts /etc/ansible/deployment.yml'"
                    '''
                  }
                  // Kill the SSM session after deploy
                  sh 'pkill -f "aws ssm start-session"'
                }
              }
            }

        stage('check stage website availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://stage.odochidevops.space"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://stage.odochidevops.space", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The stage petclinic java application is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The stage petclinic java application appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
        // stage('Run Selenium Tests on stage') {
        //     steps {
        //         echo 'Running Selenium tests on stage...'

                // Ensure Python and pip3 exist (for Amazon Linux or RHEL)
                // sh '''
                //     if ! command -v python3 &> /dev/null; then
                //         echo "Installing Python3..."
                //         sudo yum install -y python3
                //     fi

                //     if ! command -v pip3 &> /dev/null; then
                //         echo "Installing pip3..."
                //         sudo yum install -y python3-pip
                //     fi

                //     echo "Installing Selenium test dependencies..."
                //     export PATH=$PATH:/var/lib/jenkins/.local/bin
                //     pip3 install --upgrade pip
                //     pip3 install selenium pytest pytest-html
                // '''

                // Run Selenium test
        //         sh '''
        //             echo "Executing Selenium test..."
        //             pytest tests/test_homepage.py --html=report.html -v
        //         '''
        //     }
        // }
        stage ('DAST Scan') {
          steps {
            sh '''
              chmod 777 $(pwd)
              docker run -v $(pwd):/zap/wrk/:rw -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://stage.odochidevops.space -g gen.conf -r testreport.html || true
            '''
          }
        }
        stage('Request for Approval') {
            steps {
                timeout(activity: true, time: 10) {
                    input message: 'Needs Approval ', submitter: 'admin'
                }
            }
        }
        stage ('Deploying to prod Environment') {
          steps {
              script {
                // Start SSM session to bastion with port forwarding for SSH (port 22)
                sh '''
                  aws ssm start-session \
                    --target ${BASTION_ID} \
                    --region ${AWS_REGION} \
                    --document-name AWS-StartPortForwardingSession \
                    --parameters '{"portNumber":["22"],"localPortNumber":["9999"]}' \
                    &
                  sleep 5  # Wait for port forwarding to establish
                '''
                // SSH through the tunnel to Ansible server on port 22
                sshagent(['bastion-key', 'ansible-key']) {
                  sh '''
                    ssh -o StrictHostKeyChecking=no \
                        -o ProxyCommand="ssh -W %h:%p -o StrictHostKeyChecking=no ubuntu@localhost -p 9999" \
                        ec2-user@${ANSIBLE_IP} \
                        "ansible-playbook -i /etc/ansible/prod_hosts /etc/ansible/deployment.yml"
                  '''
                }
                // Terminate the SSM session
                sh 'pkill -f "aws ssm start-session"'
              }
          }
        }
        stage('check prod website availability') {
            steps {
                 sh "sleep 90"
                 sh "curl -s -o /dev/null -w \"%{http_code}\" https://prod.odochidevops.space"
                script {
                    def response = sh(script: "curl -s -o /dev/null -w \"%{http_code}\" https://prod.odochidevops.space", returnStdout: true).trim()
                    if (response == "200") {
                        slackSend(color: 'good', message: "The prod petclinic java application is up and running with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    } else {
                        slackSend(color: 'danger', message: "The prod petclinic java application appears to be down with HTTP status code ${response}.", tokenCredentialId: 'slack')
                    }
                }
            }
        }
    }
}