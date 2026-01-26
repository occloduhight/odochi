pipeline {
    agent any

    environment {
        NEXUS_REPO = 'nexus.tundeafod.click/repository/maven-releases'
    }

    stages {

        stage('Code Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn clean verify sonar:sonar'
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
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
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
    }
}
