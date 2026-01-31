pipeline {
    agent any

    environment {
        NEXUS_URL      = 'nexus.odochidevops.space'
        NEXUS_USER     = credentials('nexus-docker-username')
        NEXUS_PASSWORD = credentials('nexus-docker-password')
        DOCKER_IMAGE   = 'nexus.odochidevops.space/repository/nexus-docker-repo/apppetclinic:2.4.2'
    }

    triggers {
        pollSCM('* * * * *')
    }

    stages {
        stage('Docker: Login, Build and Push') {
            steps {
                sh '''
                echo "$NEXUS_PASSWORD" | docker login https://$NEXUS_URL --username "$NEXUS_USER" --password-stdin
                docker build -t $DOCKER_IMAGE .
                docker push $DOCKER_IMAGE
                '''
            }
        }
    }
}
