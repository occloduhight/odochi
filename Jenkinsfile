pipeline {
    agent any

    environment {
       NEXUS_USER = credentials('nexus-username')
        NEXUS_PASSWORD = credentials('nexus-password')
        NEXUS_REPO = credentials('nexus-docker-repo')
        NVD_API_KEY= credentials('nvd-key')
        BASTION_IP = credentials('bastion-ip')
        ANSIBLE_IP = credentials('ansible-ip')
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
                dependencyCheck additionalArguments: "--nvdApiKey ${NVD_KEY}", odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build Artifact') {
            steps {
                sh 'mvn clean package -DskipTests -Dcheckstyle.skip'
            }
        }

        stage('Upload WAR to Nexus') {
            steps {
                nexusArtifactUploader(
                    artifacts: [[
                        artifactId: 'autodiscovery',
                        classifier: '',
                        file: 'target/autodiscovery.war',
                        type: 'war'
                    ]],
                    credentialsId: 'nexus-creds',
                    groupId: 'com.autodiscovery',
                    nexusUrl: 'nexus.tundeafod.click',
                    nexusVersion: 'nexus3',
                    protocol: 'https',
                    repository: 'nexus-repo',
                    version: '1.0.0'
                )
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Trivy FS Scan') {
            steps {
                retry(2) {
                    sh "trivy fs . > trivyfs.txt || true"
                }
            }
        }

        stage('Docker Login & Push') {
            steps {
                sh """
                    echo $NEXUS_PASSWORD | docker login --username $NEXUS_USER --password-stdin $NEXUS_REPO
                    docker tag ${IMAGE_NAME} $NEXUS_REPO/${IMAGE_NAME}
                    docker push $NEXUS_REPO/${IMAGE_NAME}
                """
            }
        }

        stage('Trivy Image Scan') {
            steps {
                retry(2) {
                    sh "trivy image $NEXUS_REPO/${IMAGE_NAME} > trivyimage.txt || true"
                }
            }
        }

        stage('Wait for Stage ELB') {
            steps {
                script {
                    retry(10) {
                        sleep 15
                        def status = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://stage.autodiscovery.click', returnStdout: true).trim()
                        if (status == '200') {
                            echo "Stage ELB is healthy."
                            return
                        } else {
                            echo "Stage ELB not ready yet (status: ${status}). Retrying..."
                            error("Stage ELB not ready")
                        }
                    }
                }
            }
        }

        stage('Deploy to Stage') {
            steps {
                sshagent(['ansible-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@3.8.33.146 \
                        "ansible-playbook -i /etc/ansible/stage-hosts /etc/ansible/stage-playbook.yml --extra-vars 'docker_image=${IMAGE_NAME} nexus_repo=${NEXUS_REPO}'"
                    """
                }
            }
        }

        stage('Check Stage Website') {
            steps {
                retry(3) {
                    sleep 30
                    script {
                        def status = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://stage.autodiscovery.click', returnStdout: true).trim()
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

        stage('Wait for Prod ELB') {
            steps {
                script {
                    retry(10) {
                        sleep 15
                        def status = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://prod.autodiscovery.click', returnStdout: true).trim()
                        if (status == '200') {
                            echo "Prod ELB is healthy."
                            return
                        } else {
                            echo "Prod ELB not ready yet (status: ${status}). Retrying..."
                            error("Prod ELB not ready")
                        }
                    }
                }
            }
        }

        stage('Deploy to Prod') {
            steps {
                sshagent(['ansible-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@3.8.33.146 \
                        "ansible-playbook -i /etc/ansible/prod-hosts /etc/ansible/prod-playbook.yml --extra-vars 'docker_image=${IMAGE_NAME} nexus_repo=${NEXUS_REPO}'"
                    """
                }
            }
        }

        stage('Check Prod Website') {
            steps {
                retry(3) {
                    sleep 30
                    script {
                        def status = sh(script: 'curl -s -o /dev/null -w "%{http_code}" https://prod.autodiscovery.click', returnStdout: true).trim()
                        def color = status == '200' ? 'good' : 'danger'
                        slackSend(color: color, message: "Prod app HTTP status: ${status}", tokenCredentialId: 'slack')
                        if (status != '200') { error("Prod not ready") }
                    }
                }
            }
        }

    } // end stages
} // end pipeline
