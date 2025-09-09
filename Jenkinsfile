pipeline {
    agent {
        kubernetes {
            label 'jnlp-buildah'
            yamlFile 'gfj-be/jnlp-buildah.yaml'
        }
    }

    environment {
        PATH = "/opt/maven/bin:${env.PATH}"
        SONAR_URL = 'http://sonarqube.devops-tools.svc.cluster.local:9000'
        SONAR_TOKEN = credentials('sonar-token')
        GIT_CREDENTIALS_ID = 'jenkins-token-github'
    }

    stages {
        stage('Checkout') {
            steps {
                container('jnlp') {
                    checkout scm
                }
            }
        }

        stage('Get Repo Name') {
            steps {
                container('jnlp') {
                    script {
                        def repoUrl = scm.getUserRemoteConfigs()[0].getUrl()
                        def repoName = repoUrl.tokenize('/').last().replace('.git', '').toLowerCase()
                        def deploymentRepo = repoUrl.replace('.git', '') + "-deployment.git"
                        def commitSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        env.REPO_NAME = repoName
                        env.IMAGE_NAME = "docker.io/chankyswami/${repoName}:${commitSha}"
                        env.DEPLOYMENT_REPO = deploymentRepo
                        echo "Repository Name: ${repoName}"
                        echo "Docker Image Name: ${env.IMAGE_NAME}"
                        echo "Deployment Repository: ${env.DEPLOYMENT_REPO}"
                    }
                }
            }
        }

        stage('Build Maven Project') {
            steps {
                container('jnlp') {
                    dir('gfj-be') {
                        sh '''
                            set -x
                            mvn clean package -DskipTests
                        '''
                    }
                }
            }
        }

        // stage('SonarQube Analysis') {
        //     steps {
        //         container('jnlp') {
        //             dir('gfj-be') {
        //                 withSonarQubeEnv('sonar') {
        //                     sh '''
        //                         set -x
        //                         mvn sonar:sonar \
        //                             -Dsonar.projectKey=${REPO_NAME} \
        //                             -Dsonar.host.url=${SONAR_URL} \
        //                             -Dsonar.login=${SONAR_TOKEN}
        //                     '''
        //                 }
        //             }
        //         }
        //     }
        // }

        // stage('Quality Gate') {
        //     steps {
        //         container('jnlp') {
        //             script {
        //                 timeout(time: 5, unit: 'MINUTES') {
        //                     waitForQualityGate abortPipeline: true
        //                 }
        //             }
        //         }
        //     }
        // }

        stage('Build & Push Image with Buildah') {
            steps {
                container('buildah') {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-username-password', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                        script {
                            sh '''
                                set -x
                                buildah login -u "${DOCKER_USERNAME}" -p "${DOCKER_PASSWORD}" docker.io

                                # 👇 build image using Dockerfile inside gfj-be
                                buildah bud -t ${IMAGE_NAME} -f gfj-be/Dockerfile gfj-be

                                buildah push ${IMAGE_NAME}
                            '''
                        }
                    }
                }
            }
        }

        stage('Update K8s Manifests & Push to chanky Branch') {
            steps {
                container('jnlp') {
                    withCredentials([string(credentialsId: "${GIT_CREDENTIALS_ID}", variable: 'GIT_TOKEN')]) {
                        script {
                            sh '''
                                set -x
                                DEPLOYMENT_REPO_AUTH=$(echo ${DEPLOYMENT_REPO} | sed "s|https://|https://${GIT_TOKEN}@|")
                                git clone -b main ${DEPLOYMENT_REPO_AUTH} k8s-manifests
                                cd k8s-manifests

                                # Update deployment.yaml with new image
                                sed -i 's|image: .*|image: '"${IMAGE_NAME}"'|' deployment.yaml

                                git config --global user.email "c.innovator@gmail.com"
                                git config --global user.name "chankyswami"

                                git add .
                                git commit -m "Update deployment image to ${IMAGE_NAME}"
                                git push origin main
                            '''
                        }
                    }
                }
            }
        }
    }
}
