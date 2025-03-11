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
        stage('Check Modified Files') {
            steps {
                script {
                    // 단순히 src 폴더의 변경사항만 확인
                    def changedFiles = sh(returnStdout: true, script: 'git diff --name-only HEAD~1 HEAD').trim()
                    
                    if (changedFiles.contains('src/')) {
                        echo "Changes detected in src/. Proceeding with the pipeline."
                    } else {
                        echo "No changes in src/. Stopping pipeline execution."
                        currentBuild.result = 'NOT_BUILT'
                        return
                    }
                }
            }
        }


        stage('Clone Repository') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }        
            steps {
                checkout scm
            }
        }
        
        stage('Maven Build') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                withMaven(maven: 'Maven') {
                    sh 'mvn package -DskipTests'
                }
            }
        }
        
        stage('Docker Build') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    image = docker.build("${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}")
                }
            }
        }
        
        stage('Push to ACR') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
            steps {
                script {
                    sh "az acr login --name ${REGISTRY.split('\\.')[0]}"
                    sh "docker push ${REGISTRY}/${IMAGE_NAME}:v${env.BUILD_NUMBER}"
            
                }
            }
        }

        stage('CleanUp Images') {
            when {
                expression {
                    currentBuild.result != 'NOT_BUILT'
                }
            }
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
                        // Git 설정에 실제 이메일 주소 사용
                        sh "git config --global user.email '${GIT_USER_EMAIL}'"
                        sh "git config --global user.name 'Jenkins CI'"
                        
                        // 임시 디렉토리 생성 및 이동
                        sh 'rm -rf repo || true'
                        
                        // 보안 경고를 피하기 위해 따옴표 처리 수정
                        withEnv(["GITHUB_URL=https://${GIT_USERNAME}:${GIT_PASSWORD}@${GITHUB_REPO}"]) {
                            sh 'git clone "${GITHUB_URL}" repo'
                        }
                        
                        // 파일 복사 및 커밋
                        sh '''
                            cp azure/deploy.yaml repo/azure/deploy.yaml
                            cd repo
                            git add azure/deploy.yaml
                            git commit -m "Update deploy.yaml with build ${BUILD_NUMBER}" || true
                            git push origin ${GITHUB_BRANCH}
                            cd ..
                            rm -rf repo
                        '''
                    }
                }
            }
        } 
    }
}
