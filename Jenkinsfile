pipeline { agent any

environment {
    REGISTRY = 'user16s.azurecr.io'
    IMAGE_NAME = 'delivery'
    AKS_CLUSTER = 'user16-aks'
    RESOURCE_GROUP = 'user16-rsrcgrp'
    AKS_NAMESPACE = 'default'
    AZURE_CREDENTIALS_ID = 'Azure-Cred'
    TENANT_ID = 'f46af6a3-e73f-4ab2-a1f7-f33919eda5ac' // Service Principal 등록 후 생성된 ID
    GIT_USER_NAME = 'jinjoo-lab'
    GIT_USER_EMAIL = 'drasgon@naver.com'
    GITHUB_CREDENTIALS_ID = 'Github-Cred'
    GITHUB_REPO = 'github.com/jinjoo-lab/reqres_delivery'
    GITHUB_BRANCH = 'master' // 업로드할 브랜치
}

    stages {
        stage('Clone Repository') {
            steps {
                checkout scm
            }
        }
        
        stage('Maven Build') {
            steps {
                withMaven(maven: 'Maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    image = docker.build("${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Push to ACR') {
            steps {
                script {
                    sh "az acr login --name ${REGISTRY.split('\\.')[0]}"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}"
            
                }
            }
        }

        stage('CleanUp Images') {
            steps {
                sh """
                docker rmi ${REGISTRY}/${IMAGE_NAME}:v$BUILD_NUMBER
                """
            }
        }
        
        stage('Update deploy.yaml') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    sh """
                    sed -i 's|image: \"${REGISTRY}/${IMAGE_NAME}:.*\"|image: \"${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_ID}\"|' azure/deploy.yaml
                    cat azure/deploy.yaml
                    """
                }
            }
        }
        
        stage('Commit and Push to GitHub') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: GITHUB_CREDENTIALS_ID, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh """
                            git config --global user.email "your-email@example.com"
                            git config --global user.name "Jenkins CI"
                            rm -rf repo
                            git clone https://github.com/jinjoo-lab/reqres_delivery repo
                            cp azure/deploy.yaml repo/azure/deploy.yaml
                            cd repo
                            git add azure/deploy.yaml
                            git commit -m "Update deploy.yaml with build ${env.BUILD_NUMBER}"
                            git push origin ${GITHUB_BRANCH}
                            cd ..
                            rm -rf repo
                        """
                    }
                }
            }
        } 
    }
}
