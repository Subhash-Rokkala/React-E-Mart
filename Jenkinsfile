pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }
    environment {
    SONARQUBE_ENV = 'sq'
    }


    stages {

        stage('Checkout') {
            steps {
               checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'GitCreds', url: 'https://github.com/Subhash-Rokkala/React-E-Mart.git']])
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Build Project') {
            steps {
                sh 'CI=false npm run build || echo "No build step, skipping..."'
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

        stage('JENKINS TO NEXUS') {
            steps {
              withMaven(globalMavenSettingsConfig: 'settings.xml', jdk: 'jkd17', traceability: true) {
             sh 'mvn deploy'
             }
            }
        }      

        stage('Docker Build') {
            steps {
                sh 'docker build -t e-mart .'
            }
        }

        stage('Docker Run') {
        steps {
            sh 'docker stop e-mart-container || true'
            sh 'docker rm e-mart-container || true'
            sh 'docker run -d -p 8086:80 --name e-mart-container e-mart'
       }
    }
    }
}
