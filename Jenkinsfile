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
        // Forces Windows paths into the pipeline execution context to fix the "cmd" error
        PATH = "C:\\Windows\\System32;C:\\Windows;C:\\Windows\\System32\\Wbem;${env.PATH}"
        
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
                    // Replaced Linux commands with Windows Batch syntax
                    bat '@echo off && echo %DOCKER_PASS% | docker login --username %DOCKER_USER% --password-stdin'
                    
                    script {
                        docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push()
                    }
                    
                    bat 'docker logout'
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
                    // Capture previous revision using Windows Batch format
                    def previousRevision = bat(
                        script: """
                            @echo off
                            kubectl get deployment ${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} -o jsonpath="{.metadata.annotations.deployment\\.kubernetes\\.io/revision}" 2>NUL || echo 0
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
                        // Creates registry secret safely using Windows command formatting
                        bat """
                            @echo off
                            kubectl create secret docker-registry dockerhub-pull --docker-server=https://docker.io --docker-username="%DOCKER_USER%" --docker-password="%DOCKER_PASS%" --namespace="${K8S_NAMESPACE}" --dry-run=client -o yaml > secret.yaml
                            kubectl apply -f secret.yaml
                            del secret.yaml
                        """
                    }

                    bat """
                        kubectl patch deployment ${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} --type=strategic -p "{\\\"spec\\\":{\\\"template\\\":{\\\"spec\\\":{\\\"imagePullSecrets\\\":[{\\\"name\\\":\\\"dockerhub-pull\\\"}]}}}}"
                    """

                    try {
                        bat """
                            kubectl set image deployment/${K8S_DEPLOYMENT} surplus-food-portal=${IMAGE_NAME}:${IMAGE_TAG} -n ${K8S_NAMESPACE}
                            kubectl rollout status deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} --timeout=120s
                        """
                        echo 'Kubernetes deployment successful'

                    } catch (Exception e) {
                        echo 'Deployment failed. Attempting automatic rollback.'

                        if (!previousRevision.isInteger() || previousRevision == '0') {
                            error('No previous stable revision available')
                        }

                        bat """
                            kubectl rollout undo deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} --to-revision=${previousRevision}
                            kubectl rollout status deployment/${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE} --timeout=120s
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
                bat """
                    kubectl get deployment ${K8S_DEPLOYMENT} -n ${K8S_NAMESPACE}
                    kubectl get pods -n ${K8S_NAMESPACE} -l app=${K8S_DEPLOYMENT}
                    kubectl get service surplus-food-service -n ${K8S_NAMESPACE}
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
