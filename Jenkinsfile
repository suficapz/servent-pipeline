pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'pipeline-servent'
        SONAR_SERVER_NAME     = 'sonarqube-env' 
        SONAR_SCANNER_TOOL    = 'sonar-scanner'     
        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/suficapz/servent-pipeline.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"
                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh "${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=pipeline-servent-key \
                            -Dsonar.projectName='pipeline-servent-project' \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=*/venv/,/.venv/,/_pycache_/,/.pytest_cache/,/static/*"
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin
                            
                            docker build -t ${DOCKER_IMAGE}:${env.BUILD_NUMBER} -t${DOCKER_IMAGE}:latest .
                            
                            docker push ${DOCKER_IMAGE}:${env.BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE}:latest
                        """
                    }
                }
            }
        }

        stage('Update Manifest Repo') {
            steps {
                dir('manifest-workspace') {
                    sh "git clone ${MANIFEST_REPO} . || true"
                    withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS_ID}", usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
                        sh """
                            git remote set-url origin https://\${GH_USER}:\${GH_TOKEN}@github.com/suficapz/servent-pipeline.git
                            git pull --rebase origin ${MANIFEST_BRANCH}
                            
                            sed -i '/name: pipeline-servent/{n;s|image:.*|image: ${DOCKER_IMAGE}:${env.BUILD_NUMBER}|}' k8s/Deployment.yaml
                            
                            git config user.name "suficapz"
                            git config user.email "sufiyanathikal@gmail.com"
                            git commit -am "chore(manifest): update pipeline-servent image tag to ${env.BUILD_NUMBER} [skip ci]"
                            git push origin ${MANIFEST_BRANCH}
                        """
                    }
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline executed successfully! pipeline-servent image built, pushed, and manifest repo updated.'
        }
        failure {
            echo 'Pipeline failed. Please review the stage logs above.'
        }
    }
}
