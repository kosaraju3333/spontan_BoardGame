pipeline {
    
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    
    environment {
        SONAR_HOME= tool 'sonar-scanner'
    }
    
    parameters{

        // choice(name: 'action', choices: 'create\ndelete', description: 'Choose create/Destroy')
        string(name: 'ImageName', description: "name of the docker build Image", defaultValue: 'spontan-boardgame-app')
        string(name: 'ImageTag', description: "tag of the docker build Image", defaultValue: 'V1')
        string(name: 'DockerHubUser', description: "name of the DockerHub user", defaultValue: 'kosaraju333')
    }

    
    stages {
        
        stage('GIT Checkout') {
            steps {
                git credentialsId: 'GitHub-credentials', url: 'https://github.com/kosaraju3333/spontan_BoardGame.git'
            }   
        }
        
        stage('Clean') {
            steps {
                sh "mvn clean"
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('File System Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame -Dsonar.java.binaries=.'''
                }
            }
        }
        
        stage('Quality Gates') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Publish Artifacts to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker-HUB', toolName: 'docker') {
                        sh "docker build -t ${DockerHubUser}/${ImageName}:${ImageTag} ."
                    }
                }
            }
        }
        
        stage('Docker Image Scane') {
            steps {
                sh "trivy image --format table -o trivy-fs-report.html ${DockerHubUser}/${ImageName}:${ImageTag}"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'Docker-HUB', toolName: 'docker') {
                        sh "docker push ${DockerHubUser}/${ImageName}:${ImageTag}"
                    }
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                dir('eks/boardgame-app') {
                   withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'spontan-Boardgame-app', contextName: '', credentialsId: 'jenkins-eks-access-token', namespace: 'boardgame', serverUrl: 'https://FA4B52C640A3B2FC69BEE85D6CA54AB8.gr7.us-east-1.eks.amazonaws.com']]) {
                        sh "kubectl apply -f deployment.yml -n boardgame"
                        sh "kubectl apply -f LoadBalancer-service.yml -n boardgame"
                    } 
                }
                // withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'spontan-Boardgame-app', contextName: '', credentialsId: 'jenkins-eks-access-token', namespace: 'boardgame', serverUrl: 'https://FA4B52C640A3B2FC69BEE85D6CA54AB8.gr7.us-east-1.eks.amazonaws.com']]) {
                //     sh "kubectl apply -f deployment-service.yaml -n boardgame"
                // }
            }
        }
        
        stage('Verifying EKS deployments') {
            steps {
                withKubeCredentials(kubectlCredentials: [[caCertificate: '', clusterName: 'spontan-Boardgame-app', contextName: '', credentialsId: 'jenkins-eks-access-token', namespace: 'boardgame', serverUrl: 'https://FA4B52C640A3B2FC69BEE85D6CA54AB8.gr7.us-east-1.eks.amazonaws.com']]) {
                    sh "kubectl get pods -n boardgame"
                    sh "kubectl get service -n boardgame"
                }
            }
        }
    }
        
        
        
    post {
        always {
            slackSend channel: 'jenkins-production-builds', message: '@channel \n Status - ${currentBuild.currentResult} ${env.JOB_NAME} ${env.BUILD_NUMBER} ${env.BUILD_URL}'
        }
    }
    
}
