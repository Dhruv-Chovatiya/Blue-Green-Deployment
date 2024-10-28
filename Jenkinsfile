pipeline {
    agent any
    
    tools {
        maven 'maven3'
    }
    
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    
    environment {
        IMAGE_NAME = "dhruvchovatiya63907/bankapp"
        TAG = "${params.DOCKER_TAG}"
        SCANNER_HOME = tool 'sonar-scanner'
        KUBE_NAMESPACE = 'webapps'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Dhruv-Chovatiya/Blue-Green-Deployment.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        
        // stage('Trivy FS Scan') {
        //     steps {
        //         sh "trivy fs --format table -o fs.html ."
        //     }
        // }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=mutlitier -Dsonar.projectName=mutlitier -Dsonar.java.binaries=target"
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        
        stage('Publish Artifact to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-settings', maven: 'maven3', traceability: true) {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                    }
                }
            }
        }
        
        stage('Docker Push Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }
        
        // stage('Run Docker Container') {
        //     steps {
        //         script {
        //           sh "docker run -d --name bankapp-${params.DEPLOY_ENV} -p 8085:8085 ${IMAGE_NAME}:${TAG}"
        //         }
        //     }
        // }
        
        stage('Deploy MySQL Deployment and Service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'dhruv-aks', contextName: '', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, restrictKubeConfigAccess: false, serverUrl: 'dhruv-aks-dns-gs2zvjcf.hcp.centralindia.azmk8s.io') {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }
        
        stage('Deploy Service for App') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'dhruv-aks', contextName: '', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, restrictKubeConfigAccess: false, serverUrl: 'dhruv-aks-dns-gs2zvjcf.hcp.centralindia.azmk8s.io') {
                        sh """
                          if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                              kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                          fi
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml'
                    withKubeConfig(caCertificate: '', clusterName: 'dhruv-aks', contextName: '', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, restrictKubeConfigAccess: false, serverUrl: 'dhruv-aks-dns-gs2zvjcf.hcp.centralindia.azmk8s.io') {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }
        
        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'dhruv-aks', contextName: '', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, restrictKubeConfigAccess: false, serverUrl: 'dhruv-aks-dns-gs2zvjcf.hcp.centralindia.azmk8s.io') {
                        sh """
                          kubectl patch service bankapp-service -p "{\\"spec\\": {\\"selector\\": {\\"app\\": \\"bankapp\\", \\"version\\": \\"${newEnv}\\"}}}" -n ${KUBE_NAMESPACE}
                        """
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'dhruv-aks', contextName: '', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, restrictKubeConfigAccess: false, serverUrl: 'dhruv-aks-dns-gs2zvjcf.hcp.centralindia.azmk8s.io') {
                        sh """
                          kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                          kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
    }
}
