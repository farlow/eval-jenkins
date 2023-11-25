pipeline {
environment { // Declaration of environment variables
DOCKER_ID="farloche" // replace this with your docker-id
DOCKER_CAST_IMAGE="cast-service"
DOCKER_MOVIE_IMAGE="movie-service"
DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
}
agent any // Jenkins will be able to select all available agents
stages {
        stage(' Docker Build'){ // docker build image stage
            steps {
                script {
                sh '''
                 docker rm -f nginx movie_db cast_db movie_service cast_service
                 docker build -t $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG ./cast-service
                 docker build -t $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG ./movie-service
                '''
                }
            }
        }
        stage('Docker run'){ // run container from our builded image
                steps {
                    script {
                    sh '''
                    docker run -d --name movie-db -v postgres_data_movie:/var/lib/postgresql/data/ -e POSTGRES_USER=movie_db_username -e POSTGRES_PASSWORD=movie_db_password -e POSTGRES_DB=movie_db_dev -d postgres:12.1-alpine
                    docker run -d --name cast-db -v postgres_data_cast:/var/lib/postgresql/data/ -e POSTGRES_USER=cast_db_username -e POSTGRES_PASSWORD=cast_db_password -e POSTGRES_DB=cast_db_dev -d postgres:12.1-alpine
                    docker run -d --name movie-service -p 8001:8000 -e DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie-db/movie_db_dev -e CAST_SERVICE_HOST_URL=http://cast_service:8000/api/v1/casts/ --link movie-db $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                    docker run -d --name cast-service -p 8002:8000 -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast-db/cast_db_dev --link cast-db $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                    docker run -d --name nginx -p 8081:8080 -v ./nginx_config.conf:/etc/nginx/conf.d/default.conf --link cast-service --link movie-service -d nginx:latest
                    '''
                    }
                }
            }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl localhost:8081
                    '''
                    }
            }

        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG

                '''
                }
            }

        }

stage('Deploiement en dev'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp helm/values.yaml values.yml
                cat values.yml
                sed -i "s+name.*+name: movie-db+g" values.yml                
                helm upgrade --install movie-db --values=values.yml ./helm-db/ --namespace=dev
                sed -i "s+name.*+name: cast-db+g" values.yml
                helm upgrade --install cast-db --values=values.yml ./helm-db/ --namespace=dev
                '''
                }
            }

        }
stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp helm/values.yaml values.yml
                cat values.yml                
                helm install movie-db --values=values.yml .
                helm upgrade --install movie-db --values=values.yml .


                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app fastapi --values=values.yml --namespace staging
                '''
                }
            }

        }
  stage('Deploiement en prod'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
            // Create an Approval Button with a timeout of 15minutes.
            // this require a manuel validation in order to deploy on production environment
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }

                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app fastapi --values=values.yml --namespace prod
                '''
                }
            }

        }

}
}
