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
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
  - name: aws-cli
    image: amazon/aws-cli
    command: ["sleep"]
    args: ["9999999"]
    volumeMounts:
    - name: docker-config
      mountPath: /root/.docker
  - name: jgit
    image: alpine/git
    command: ["sleep"]
    args: ["9999999"]
  volumes:
  - name: docker-config
    emptyDir: {}
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
                // Крок 1: Отримуємо токен через контейнер з AWS CLI
                container('aws-cli') {
                    sh """
                        TOKEN=\$(aws ecr get-login-password --region ${REGION})
                        echo "{\\"auths\\":{\\"${ECR_REPO.split('/')[0]}\\":{\\"auth\\":\\"\$(echo -n AWS:\$TOKEN | base64 | tr -d '\n')\\"}}}" > /root/.docker/config.json
                    """
                }
                // Крок 2: Kaniko використовує готовий config.json для авторизації
                container('kaniko') {
                    sh "/kaniko/executor --context ${WORKSPACE} --dockerfile ${WORKSPACE}/Dockerfile --destination ${ECR_REPO}:${BUILD_NUMBER}"
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
                            
                            # Клонуємо інфраструктурний репозиторій
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/goit-devops-CI-CD.git infra-repo
                            cd infra-repo
                            git checkout lesson-8-9

                            # Оновлюємо тег у values.yaml
                            sed -i "s/tag: .*/tag: ${BUILD_NUMBER}/g" lesson-8-9/charts/django-app/values.yaml
                            
                            # Пушимо зміни
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