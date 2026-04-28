pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        SONARQUBE_ENV = 'sq'
        NEXUS_URL = 'http://3.89.218.67:8081'
        NEXUS_REPO = 'react-artifacts'
        DOCKER_IMAGE = 'e-mart'
        DOCKER_TAG = "${BUILD_NUMBER}"
        AWS_REGION = 'ap-south-1'
        CLUSTER_NAME = 'mycluster'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        credentialsId: 'GitCreds',
                        url: 'https://github.com/Subhash-Rokkala/React-E-Mart.git'
                    ]]
                )
            }
        }

        stage('Install & Build') {
            steps {
                sh '''
                npm install
                CI=false npm run build
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                    npx sonar-scanner \
                    -Dsonar.projectKey=e-mart \
                    -Dsonar.sources=src
                    '''
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

        stage('Upload Artifact to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS'
                )]) {
                    sh '''
                    zip -r dist.zip dist/

                    curl -u $USER:$PASS \
                    --upload-file dist.zip \
                    $NEXUS_URL/repository/$NEXUS_REPO/dist-${BUILD_NUMBER}.zip
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $DOCKER_IMAGE:$DOCKER_TAG .
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'DockerCred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin

                    docker tag $DOCKER_IMAGE:$DOCKER_TAG $DOCKER_USER/$DOCKER_IMAGE:$DOCKER_TAG
                    docker push $DOCKER_USER/$DOCKER_IMAGE:$DOCKER_TAG
                    '''
                }
            }
        }

      
        stage('Update kubeconfig') {
            steps {
                sh '''
                aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME
                '''
            }
        }

        
        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                kubectl config current-context
                kubectl get nodes
                kubectl apply -f k8s/Deployment.yml
                kubectl apply -f k8s/loadBalancerService.yml
                '''
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
