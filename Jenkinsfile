pipeline {
    environment {
        DOCKER_ID = "killuazoaldyek"
        MOVIE_SERVICE_IMAGE = "movie-service"
        CAST_SERVICE_IMAGE = "cast-service"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        COMMIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
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
                        echo "DEBUG - COMMIT SHA : ${COMMIT_SHA}"
                        
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        
                        # Tag et push avec le tag de version
                        docker push $DOCKER_ID/$MOVIE_SERVICE_IMAGE:$DOCKER_TAG
                        docker push $DOCKER_ID/$CAST_SERVICE_IMAGE:$DOCKER_TAG
                        
                        # Tag et push avec le tag latest
                        docker tag $DOCKER_ID/$MOVIE_SERVICE_IMAGE:$DOCKER_TAG $DOCKER_ID/$MOVIE_SERVICE_IMAGE:latest
                        docker tag $DOCKER_ID/$CAST_SERVICE_IMAGE:$DOCKER_TAG $DOCKER_ID/$CAST_SERVICE_IMAGE:latest
                        
                        # Tag et push avec le SHA du commit
                        docker tag $DOCKER_ID/$MOVIE_SERVICE_IMAGE:$DOCKER_TAG $DOCKER_ID/$MOVIE_SERVICE_IMAGE:${COMMIT_SHA}
                        docker tag $DOCKER_ID/$CAST_SERVICE_IMAGE:$DOCKER_TAG $DOCKER_ID/$CAST_SERVICE_IMAGE:${COMMIT_SHA}
                        docker push $DOCKER_ID/$MOVIE_SERVICE_IMAGE:${COMMIT_SHA}
                        docker push $DOCKER_ID/$CAST_SERVICE_IMAGE:${COMMIT_SHA}
                        
                        # Push des tags latest
                        docker push $DOCKER_ID/$MOVIE_SERVICE_IMAGE:latest
                        docker push $DOCKER_ID/$CAST_SERVICE_IMAGE:latest
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
                        
                        # Nettoyage des ressources existantes
                        echo "Nettoyage des ressources dans le namespace dev..."
                        kubectl delete deployment -l app.kubernetes.io/instance=movie-cast -n dev --ignore-not-found=true
                        kubectl delete service -l app.kubernetes.io/instance=movie-cast -n dev --ignore-not-found=true
                        kubectl delete ingress -l app.kubernetes.io/instance=movie-cast -n dev --ignore-not-found=true
                        
                        # Attente que tous les pods soient supprimés
                        echo "Attente de la suppression des pods..."
                        while kubectl get pods -n dev -l app.kubernetes.io/instance=movie-cast 2>/dev/null | grep -q .; do
                            echo "Pods en cours de suppression..."
                            sleep 2
                        done
                        echo "Tous les pods ont été supprimés"
                        
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
                        
                        # Nettoyage des ressources existantes
                        echo "Nettoyage des ressources dans le namespace staging..."
                        kubectl delete deployment -l app.kubernetes.io/instance=movie-cast -n staging --ignore-not-found=true
                        kubectl delete service -l app.kubernetes.io/instance=movie-cast -n staging --ignore-not-found=true
                        kubectl delete ingress -l app.kubernetes.io/instance=movie-cast -n staging --ignore-not-found=true
                        
                        # Attente que tous les pods soient supprimés
                        echo "Attente de la suppression des pods..."
                        while kubectl get pods -n staging -l app.kubernetes.io/instance=movie-cast 2>/dev/null | grep -q .; do
                            echo "Pods en cours de suppression..."
                            sleep 2
                        done
                        echo "Tous les pods ont été supprimés"
                        
                        kubectl get namespace staging || kubectl create namespace staging
                        sed -i "s|tag:.*|tag: ${DOCKER_TAG}|g" movie-cast/environments/values-staging.yaml
                        helm upgrade --install movie-cast ./movie-cast --values=movie-cast/environments/values-staging.yaml --namespace staging
                    '''
                }
            }
        }
        
        stage('Deploiement en prod') {
            when {
                expression { 
                    return env.GIT_BRANCH == 'origin/master'
                }
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
                        
                        # Nettoyage des ressources existantes
                        echo "Nettoyage des ressources dans le namespace prod..."
                        kubectl delete deployment -l app.kubernetes.io/instance=movie-cast -n prod --ignore-not-found=true
                        kubectl delete service -l app.kubernetes.io/instance=movie-cast -n prod --ignore-not-found=true
                        kubectl delete ingress -l app.kubernetes.io/instance=movie-cast -n prod --ignore-not-found=true
                        
                        # Attente que tous les pods soient supprimés
                        echo "Attente de la suppression des pods..."
                        while kubectl get pods -n prod -l app.kubernetes.io/instance=movie-cast 2>/dev/null | grep -q .; do
                            echo "Pods en cours de suppression..."
                            sleep 2
                        done
                        echo "Tous les pods ont été supprimés"
                        
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