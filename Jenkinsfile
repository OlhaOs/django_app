pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ["sleep"]
    args: ["9999999"]
  - name: jgit
    image: alpine/git
    command: ["sleep"]
    args: ["9999999"]
"""
        }
    }
    environment {
        ECR_REPO = "482745810990.dkr.ecr.us-west-2.amazonaws.com/django-app"
        REGION   = "us-west-2"
    }
    stages {
        stage('Build and Push to ECR') {
            steps {
                container('kaniko') {
                    sh """
                        # Отримуємо токен авторизації AWS та записуємо його у форматі Docker
                        # Ми використовуємо стандартний вивід aws ecr get-login-password
                        
                        export AWS_REGION=${REGION}
                        TOKEN=\$(aws ecr get-login-password --region ${REGION})
                        AUTH=\$(echo -n "AWS:\$TOKEN" | base64 | tr -d '\n')
                        
                        mkdir -p /kaniko/.docker
                        echo "{\"auths\":{\"${ECR_REPO.split('/')[0]}\":{\"auth\":\"\$AUTH\"}}}" > /kaniko/.docker/config.json
                        
                        # Запускаємо збірку
                        /kaniko/executor --context ${WORKSPACE} \
                                         --dockerfile ${WORKSPACE}/Dockerfile \
                                         --destination ${ECR_REPO}:${BUILD_NUMBER}
                    """
                }
            }
        }
        stage('Update Helm Tag in Git') {
            steps {
                container('jgit') {
                    withCredentials([usernamePassword(credentialsId: 'github-token', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                            git config --global user.email "jenkins@example.com"
                            git config --global user.name "Jenkins CI"
                            
                            # 1. Клонуємо інфраструктурний репозиторій
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/goit-devops-CI-CD.git infra-repo
                            cd infra-repo
                            git checkout lesson-8-9

                            # 2. Оновлюємо тег (враховуємо шлях lesson-8-9)
                            sed -i "s/tag: .*/tag: ${BUILD_NUMBER}/g" lesson-8-9/charts/django-app/values.yaml
                            
                            # 3. Пушимо зміни
                            git add lesson-8-9/charts/django-app/values.yaml
                            git commit -m "Update image tag to ${BUILD_NUMBER} [skip ci]"
                            git push origin lesson-8-9
                        """
                    }
                }
            }
        }
    }
}