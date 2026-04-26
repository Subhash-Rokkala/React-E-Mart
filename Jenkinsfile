pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        SONARQUBE_ENV = 'sq'
        NEXUS_URL = 'http://3.86.224.143:8081'
        NEXUS_REPO = 'react-artifacts'
        DOCKER_IMAGE = 'e-mart'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scmGit(
                    branches: [[name: '*/main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        credentialsId: 'GitCreds',
                        url: 'https://github.com/Subhash-Rokkala/React-E-Mart.git'
                    ]]
                )
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build Project') {
            steps {
                sh 'CI=false npm run build'
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
                    npm install
                    CI=false npm run build
                    zip -r dist.zip dist/

                    curl -u $USER:$PASS \
                    --upload-file dist.zip \
                    $NEXUS_URL/repository/$NEXUS_REPO/dist.zip
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t $DOCKER_IMAGE ."
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

                    docker tag e-mart $DOCKER_USER/e-mart:latest
                    docker push $DOCKER_USER/e-mart:latest
                    '''
                }
            }
        }

        stage('Docker Run') {
            steps {
                sh '''
                docker stop e-mart-container || true
                docker rm e-mart-container || true
                docker run -d -p 8086:80 --name e-mart-container e-mart
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                    kubectl apply -f k8s/Deployment.yml
                    kubectl apply -f k8s/loadBalancerService.yml
                    '''
                }
            }
        }
    }
}
