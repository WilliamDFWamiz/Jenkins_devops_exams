pipeline {
    environment {
        DOCKER_ID = "killuazoaldyek"
        MOVIE_SERVICE_IMAGE = "movie-service"
        CAST_SERVICE_IMAGE = "cast-service"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }
    
    agent any
    
    stages {
        stage('Vérification des credentials') {
            steps {
                script {
                    try {
                        withCredentials([file(credentialsId: 'config', variable: 'KUBECONFIG')]) {
                            sh 'echo "Credentials Kubernetes trouvés"'
                        }
                    } catch (Exception e) {
                        error """
                            ERREUR CRITIQUE : Les credentials Kubernetes sont manquants !
                            Configuration requise dans Jenkins :
                            1. 'Manage Jenkins' > 'Credentials' > 'System' > 'Global credentials'
                            2. 'Add Credentials'
                            3. 'Kubernetes configuration (kubeconfig)'
                            4. ID: config
                            5. Configuration Kubernetes
                        """
                    }
                }
            }
        }

        stage('Build Services') {
            parallel {
                stage('Build Movie Service') {
                    steps {
                        script {
                            sh '''
                                docker rm -f movie-service || true
                                docker build -t $DOCKER_ID/$MOVIE_SERVICE_IMAGE:$DOCKER_TAG ./movie-service
                            '''
                        }
                    }
                }
                
                stage('Build Cast Service') {
                    steps {
                        script {
                            sh '''
                                docker rm -f cast-service || true
                                docker build -t $DOCKER_ID/$CAST_SERVICE_IMAGE:$DOCKER_TAG ./cast-service
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                        echo "DEBUG - Branche actuelle : $(git branch --show-current)"
                        echo "DEBUG - GIT_BRANCH : ${GIT_BRANCH}"
                        echo "DEBUG - BRANCH_NAME : ${BRANCH_NAME}"
                        
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker push $DOCKER_ID/$MOVIE_SERVICE_IMAGE:$DOCKER_TAG
                        docker push $DOCKER_ID/$CAST_SERVICE_IMAGE:$DOCKER_TAG
                    '''
                }
            }
        }
        
        stage('Deploiement en dev') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace dev || kubectl create namespace dev
                        sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" movie-cast/environments/values-dev.yaml
                        helm upgrade --install movie-cast ./movie-cast --values=movie-cast/environments/values-dev.yaml --namespace dev
                    '''
                }
            }
        }
        
        stage('Deploiement en staging') {
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace staging || kubectl create namespace staging
                        
                        # Mise à jour du tag de manière plus sûre
                        sed -i "s|tag:.*|tag: ${DOCKER_TAG}|g" movie-cast/environments/values-staging.yaml
                        
                        # Vérification du fichier YAML
                        cat movie-cast/environments/values-staging.yaml
                        
                        helm upgrade --install movie-cast ./movie-cast --values=movie-cast/environments/values-staging.yaml --namespace staging
                    '''
                }
            }
        }
        
        stage('Deploiement en prod') {
            when {
                branch 'master'
            }
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    sh '''
                        echo "Branche actuelle : $(git branch --show-current)"
                        echo "BRANCH_NAME: ${BRANCH_NAME}"
                        echo "GIT_BRANCH: ${GIT_BRANCH}"
                        echo "Vérification de la condition de branche..."
                    '''
                }
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production ?', ok: 'Yes'
                }
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace prod || kubectl create namespace prod
                        sed -i "s|tag:.*|tag: ${DOCKER_TAG}|g" movie-cast/environments/values-prod.yaml
                        helm upgrade --install movie-cast ./movie-cast --values=movie-cast/environments/values-prod.yaml --namespace prod
                    '''
                }
            }
        }
    }
    
    post {
        failure {
            echo "This will run if the job failed"
            mail to: "william.dalfarra1@gmail.com",
                 subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
                 body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
        }
        always {
            script {
                sh '''
                    # Nettoyage des conteneurs arrêtés
                    docker container prune -f
                    
                    # Nettoyage des images non utilisées
                    docker image prune -f
                    
                    # Nettoyage des images plus anciennes que 24h
                    docker image prune -a --filter "until=24h" -f
                '''
            }
        }
    }
}