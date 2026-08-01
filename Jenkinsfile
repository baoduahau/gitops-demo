pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: docker
      image: docker:27-cli
      command:
        - sleep
      args:
        - 99d
      tty: true
      securityContext:
        runAsUser: 0
      volumeMounts:
        - name: docker-sock
          mountPath: /var/run/docker.sock
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
        type: Socket
'''
        }
    }

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    environment {
        IMAGE = "docker.io/baoduahau/gitops-demo"
        TAG = "${BUILD_NUMBER}"
        SKIP_CI = "false"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/baoduahau/gitops-demo.git'
            }
        }

        stage('Check Commit') {
            steps {
                script {
                    def message = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()

                    echo "Commit message: ${message}"

                    if (message.contains('[skip ci]')) {
                        env.SKIP_CI = 'true'
                        currentBuild.description =
                            'GitOps write-back commit skipped'
                    }
                }
            }
        }

        stage('Docker Check') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }
            steps {
                container('docker') {
                    sh 'docker version'
                }
            }
        }

        stage('Build Image') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }
            steps {
                container('docker') {
                    sh 'docker build -t "$IMAGE:$TAG" .'
                }
            }
        }

        stage('Push Image') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                            set +x
                            echo "$DOCKER_PASS" |
                              docker login \
                                -u "$DOCKER_USER" \
                                --password-stdin
                            set -x

                            docker push "$IMAGE:$TAG"
                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Update Helm') {
            when {
                expression {
                    env.SKIP_CI != 'true'
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'GITHUB_USER',
                    passwordVariable: 'GITHUB_TOKEN'
                )]) {
                    sh '''
                        sed -i \
                          "s/^  tag:.*/  tag: \\"${TAG}\\"/" \
                          chart/values.yaml

                        git config user.email "jenkins@lab.local"
                        git config user.name "jenkins"

                        git add chart/values.yaml
                        git commit \
                          -m "Update image ${TAG} [skip ci]"

                        set +x
                        git push \
                          "https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/baoduahau/gitops-demo.git" \
                          HEAD:main
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
            echo "Image: ${IMAGE}:${TAG}"
        }

        failure {
            echo "Pipeline failed. Review Console Output."
        }
    }
}
