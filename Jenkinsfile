pipeline {
    agent any

    environment {
        NEXUS_HOST = 'nexus.tundeafod.click'
        NEXUS_IP   = '10.0.1.5'
        NEXUS_REPO = "http://${NEXUS_HOST}:8081/repository/nexus-repo"
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
                withCredentials([string(credentialsId: 'nvd-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck additionalArguments: "--nvdApiKey ${NVD_API_KEY}", odcInstallation: 'DP-Check'
                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
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
                sh 'trivy fs --insecure . > trivyfs.txt'
            }
        }

        stage('Docker Login & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-repo', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh """
                        echo \$NEXUS_PASS | docker login -u \$NEXUS_USER --password-stdin ${NEXUS_REPO} || exit 1
                        docker tag ${IMAGE_NAME} ${NEXUS_REPO}/${IMAGE_NAME} || exit 1
                        docker push ${NEXUS_REPO}/${IMAGE_NAME} || exit 1
                    """
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --insecure ${NEXUS_REPO}/${IMAGE_NAME} > trivyimage.txt"
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
                        def status = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://stage.tundeafod.click', returnStdout: true).trim()
                        def color = status == '200' ? 'good' : 'danger'
                        slackSend(color: color, message: "Stage app HTTP status: ${status}", tokenCredentialId: 'slack')
                        if (status != '200') { error("Stage not ready") }
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
                        def status = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://prod.tundeafod.click', returnStdout: true).trim()
                        def color = status == '200' ? 'good' : 'danger'
                        slackSend(color: color, message: "Prod app HTTP status: ${status}", tokenCredentialId: 'slack')
                        if (status != '200') { error("Prod not ready") }
                    }
                }
            }
        }

    } // end of stages
} // end of pipeline
