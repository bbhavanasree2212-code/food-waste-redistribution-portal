
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    parameters {
        choice(
            name: 'RELEASE_MODE',
            choices: ['STABLE', 'FAULTY_ROLLBACK_TEST'],
            description: 'Choose normal deployment or faulty rollout rollback test.'
        )
        booleanParam(
            name: 'DEPLOY_TO_KUBERNETES',
            defaultValue: true,
            description: 'Deploy and verify the app in Kubernetes.'
        )
    }

    environment {
        DOCKERHUB_USERNAME = 'bbhavanasree'
        IMAGE_NAME = 'food-waste-portal'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        KUBECONFIG_CREDENTIALS_ID = 'kubernetes-creds'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                bat 'git log -1 --oneline'
            }
        }

        stage('Parallel Validation') {
            parallel {
                stage('HTML Validation') {
                    steps {
                        bat 'findstr /I /C:"Surplus Food Listings" index.html'
                        bat 'findstr /I /C:"Redistributed" index.html'
                        bat 'findstr /I /C:"Community Meal Network" index.html'
                    }
                }

                stage('Kubernetes YAML Validation') {
                    steps {
                        bat 'findstr /C:"kind: Deployment" deployment.yaml'
                        bat 'findstr /C:"readinessProbe" deployment.yaml'
                        bat 'findstr /C:"kind: Service" deployment.yaml'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "v${env.BUILD_NUMBER}"
                    bat "docker build -t ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG} ."
                    bat "docker image inspect ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG} > image-inspect.json"
                }
            }
        }

        stage('Local Container Smoke Test') {
            steps {
                bat 'docker rm -f food-waste-smoke >NUL 2>&1 || exit /b 0'
                bat "docker run -d --name food-waste-smoke -p 18088:80 ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                bat '''
                powershell -NoProfile -ExecutionPolicy Bypass -Command "$ok=$false; for($i=0;$i -lt 15;$i++){try{$r=Invoke-WebRequest -UseBasicParsing http://localhost:18088; if($r.StatusCode -eq 200 -and $r.Content.Contains('Surplus Food Listings')){$ok=$true;break}}catch{}; Start-Sleep -Seconds 2}; if(-not $ok){docker logs food-waste-smoke; exit 1}"
                '''

                bat 'docker rm -f food-waste-smoke'
            }
        }

        stage('Archive Build Artifacts') {
            steps {
                archiveArtifacts(
                    artifacts: 'index.html,Dockerfile,Jenkinsfile,deployment.yaml,image-inspect.json',
                    fingerprint: true
                )
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    script {
                        int loginStatus = bat(
                            returnStatus: true,
                            script: '''
                            @echo off
                            powershell -NoProfile -ExecutionPolicy Bypass -Command "$env:DH_TOKEN | docker login --username $env:DH_USER --password-stdin"
                            '''
                        )

                        if (loginStatus != 0) {
                            error('Docker Hub login failed. Check dockerhub-creds.')
                        }

                        try {
                            bat "docker push ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"
                            bat "docker tag ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG} ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:latest"
                            bat "docker push ${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:latest"
                        } finally {
                            bat 'docker logout'
                        }
                    }
                }
            }
        }

        stage('Deploy and Verify on Kubernetes') {
            when {
                expression { return params.DEPLOY_TO_KUBERNETES }
            }
            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: "${KUBECONFIG_CREDENTIALS_ID}",
                            variable: 'KUBECONFIG'
                        )
                    ]) {
                        bat '''
                        powershell -NoProfile -ExecutionPolicy Bypass -Command "(Get-Content deployment.yaml -Raw).Replace('YOUR_DOCKERHUB_USERNAME',$env:DOCKERHUB_USERNAME) | Set-Content deployment-rendered.yaml"
                        '''

                        bat 'kubectl apply -f deployment-rendered.yaml'
                        bat "kubectl set image deployment/food-waste-portal food-waste-portal=${env.DOCKERHUB_USERNAME}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                        if (params.RELEASE_MODE == 'FAULTY_ROLLBACK_TEST') {
                            echo 'Applying deliberately faulty readiness probe.'

                            writeFile(
                                file: 'faulty-patch.json',
                                text: '{"spec":{"template":{"spec":{"containers":[{"name":"food-waste-portal","readinessProbe":{"httpGet":{"path":"/intentionally-missing-health-check","port":"http"},"initialDelaySeconds":1,"periodSeconds":3,"timeoutSeconds":1,"failureThreshold":2}}]}}}}'
                            )

                            bat 'kubectl patch deployment food-waste-portal --type=strategic --patch-file=faulty-patch.json'

                            int faultyStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/food-waste-portal --timeout=90s'
                            )

                            if (faultyStatus == 0) {
                                bat 'kubectl rollout undo deployment/food-waste-portal'
                                bat 'kubectl rollout status deployment/food-waste-portal --timeout=120s'
                                error('Faulty rollout unexpectedly passed.')
                            }

                            echo 'Failed rollout detected. Collecting diagnostics.'
                            bat 'kubectl get pods -l app=food-waste-portal -o wide'
                            bat 'kubectl describe deployment food-waste-portal'
                            bat 'kubectl rollout history deployment/food-waste-portal'

                            int undoStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout undo deployment/food-waste-portal'
                            )

                            if (undoStatus != 0) {
                                error('Automatic rollback command failed.')
                            }

                            int recoveryStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/food-waste-portal --timeout=120s'
                            )

                            if (recoveryStatus != 0) {
                                error('Previous stable revision did not recover.')
                            }

                            bat 'kubectl get deployment food-waste-portal'
                            bat 'kubectl get pods -l app=food-waste-portal'
                            bat 'kubectl get service food-waste-portal'

                            echo 'Rollback succeeded and the previous revision is healthy.'
                        } else {
                            int rolloutStatus = bat(
                                returnStatus: true,
                                script: 'kubectl rollout status deployment/food-waste-portal --timeout=120s'
                            )

                            if (rolloutStatus != 0) {
                                echo 'Stable rollout failed. Attempting rollback.'
                                bat 'kubectl describe deployment food-waste-portal'

                                int undoStatus = bat(
                                    returnStatus: true,
                                    script: 'kubectl rollout undo deployment/food-waste-portal'
                                )

                                if (undoStatus != 0) {
                                    error('Deployment and rollback both failed.')
                                }

                                bat 'kubectl rollout status deployment/food-waste-portal --timeout=120s'
                                error('Stable deployment failed; previous revision was restored.')
                            }

                            bat 'kubectl get deployment food-waste-portal'
                            bat 'kubectl get pods -l app=food-waste-portal'
                            bat 'kubectl get service food-waste-portal'
                            echo 'Stable deployment completed successfully.'
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            bat 'docker rm -f food-waste-smoke >NUL 2>&1 || exit /b 0'
            echo 'Pipeline finished. Review logs, artifacts, and Kubernetes status.'
        }
    }
}
