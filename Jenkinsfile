pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['development', 'production'],
            description: 'Choose the deployment environment'
        )

        booleanParam(
            name: 'PUSH_IMAGE',
            defaultValue: true,
            description: 'Push the image to Docker Hub'
        )
    }

    environment {
        IMAGE_NAME = 'sureshkrishnak/surplus-food-portal'
        IMAGE_TAG = "build-${BUILD_NUMBER}"
        K8S_DEPLOYMENT = 'surplus-food-portal'
        K8S_NAMESPACE = 'default'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Parallel Validation') {
            parallel {

                stage('Validate HTML') {
                    steps {
                        script {
                            if (!fileExists('index.html')) {
                                error('index.html is missing')
                            }

                            def html = readFile('index.html')

                            if (!html.contains('<html')) {
                                error('HTML document is invalid')
                            }

                            if (!html.contains('foodRecords')) {
                                error('Food records section is missing')
                            }

                            echo 'HTML validation passed'
                        }
                    }
                }

                stage('Validate Deployment Files') {
                    steps {
                        script {
                            ['Dockerfile', 'deployment.yaml', 'Jenkinsfile'].each {
                                if (!fileExists(it)) {
                                    error("${it} is missing")
                                }
                            }

                            echo 'Required files validated'
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Push Docker Image') {
            when {
                expression {
                    params.PUSH_IMAGE
                }
            }

            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set +x
                        printf '%s' "$DOCKER_PASS" |
                            docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin
                    '''

                    script {
                        docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push()
                    }

                    sh 'docker logout'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            when {
                expression {
                    params.DEPLOY_ENV == 'development'
                }
            }

            steps {
                script {
                    def previousRevision = sh(
                        script: """
                            kubectl get deployment ${K8S_DEPLOYMENT} \
                                -n ${K8S_NAMESPACE} \
                                -o jsonpath='{.metadata.annotations.deployment\\.kubernetes\\.io/revision}' \
                                2>/dev/null || echo 0
                        """,
                        returnStdout: true
                    ).trim()

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub-credentials',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh '''
                            set +x

                            kubectl create secret docker-registry dockerhub-pull \
                                --docker-server=https://index.docker.io/v1/ \
                                --docker-username="$DOCKER_USER" \
                                --docker-password="$DOCKER_PASS" \
                                --namespace="$K8S_NAMESPACE" \
                                --dry-run=client -o yaml |
                            kubectl apply -f -
                        '''
                    }

                    sh """
                        kubectl patch deployment ${K8S_DEPLOYMENT} \
                            -n ${K8S_NAMESPACE} \
                            --type=strategic \
                            -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"dockerhub-pull"}]}}}}'
                    """

                    try {
                        sh """
                            kubectl set image \
                                deployment/${K8S_DEPLOYMENT} \
                                surplus-food-portal=${IMAGE_NAME}:${IMAGE_TAG} \
                                -n ${K8S_NAMESPACE}

                            kubectl rollout status \
                                deployment/${K8S_DEPLOYMENT} \
                                -n ${K8S_NAMESPACE} \
                                --timeout=120s
                        """

                        echo 'Kubernetes deployment successful'

                    } catch (Exception e) {

                        echo 'Deployment failed. Attempting automatic rollback.'

                        if (!previousRevision.isInteger() ||
                            previousRevision == '0') {
                            error('No previous stable revision available')
                        }

                        sh """
                            kubectl rollout undo \
                                deployment/${K8S_DEPLOYMENT} \
                                -n ${K8S_NAMESPACE} \
                                --to-revision=${previousRevision}

                            kubectl rollout status \
                                deployment/${K8S_DEPLOYMENT} \
                                -n ${K8S_NAMESPACE} \
                                --timeout=120s
                        """

                        error('Faulty rollout detected. Previous revision restored.')
                    }
                }
            }
        }

        stage('Verify Deployment') {
            when {
                expression {
                    params.DEPLOY_ENV == 'development'
                }
            }

            steps {
                sh """
                    kubectl get deployment \
                        ${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE}

                    kubectl get pods \
                        -n ${K8S_NAMESPACE} \
                        -l app=${K8S_DEPLOYMENT}

                    kubectl get service \
                        surplus-food-service -n ${K8S_NAMESPACE}
                """
            }
        }

        stage('Archive Build Artifacts') {
            steps {
                archiveArtifacts(
                    artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                    fingerprint: true
                )
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully'
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }

        always {
            echo 'Pipeline execution finished'
        }
    }
}
