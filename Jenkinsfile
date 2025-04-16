pipeline {
environment { 
DOCKER_ID = "jclejeune71" 
DOCKER_IMAGE_CAST = "castservice"
DOCKER_IMAGE_MOVIE = "movieservice"
DOCKER_TAG = "v.${BUILD_ID}.0" 
}
agent any // Jenkins will be able to select all available agents
stages {
        stage(' Docker Build'){ // docker build image stage
            steps {
                script {
                sh '''
                 docker rm -f jenkins
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG ./cast-service
                 docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG ./movie-service				 
                sleep 6
                '''
                }
            }
        }
        stage('Docker run'){ // run container from our builded image
                steps {
                    script {
                    sh '''
                    docker run -d --name movie_db -v postgres_data_movie:/var/lib/postgresql/data/ -e POSTGRES_USER=movie_db_username -e POSTGRES_PASSWORD=movie_db_password -e POSTGRES_DB=movie_db_dev postgres:12.1-alpine
                    docker run -d --name $DOCKER_IMAGE_MOVIE --link movie_db:movie_db -v ./movie-service:/app/ -p 8001:8000 -e DATABASE_URI=postgresql://movie_db_username:movie_db_password@movie_db/movie_db_dev -e CAST_SERVICE_HOST_URL=http://cast_service:8000/api/v1/casts/ $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
                    docker run -d --name cast_db -v postgres_data_cast:/var/lib/postgresql/data/ -e POSTGRES_USER=cast_db_username -e POSTGRES_PASSWORD=cast_db_password -e POSTGRES_DB=cast_db_dev postgres:12.1-alpine
                    docker run -d --name $DOCKER_IMAGE_CAST --link cast_db:cast_db -v ./cast-service:/app/ -p 8002:8000 -e DATABASE_URI=postgresql://cast_db_username:cast_db_password@cast_db/cast_db_dev $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
                    docker run -d --name nginx --link $DOCKER_IMAGE_CAST:$DOCKER_IMAGE_CAST --link $DOCKER_IMAGE_MOVIE:$DOCKER_IMAGE_MOVIE -v ./nginx_config.conf:/etc/nginx/conf.d/default.conf -p 8080:8080 nginx:latest
					
					sleep 10
                    '''
                    }
                }
            }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    echo Acceptance
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
				docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG
				docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG
				
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
                cp charts/cast-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app ./castservice --values=values.yml --namespace dev
                cp charts/movie-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app ./movieservice --values=values.yml --namespace dev
                '''
                }
            }
    
        }
        stage('Deploiement en QA'){
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
                cp charts/cast-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app castservice --values=values.yml --namespace qa
                cp charts/movie-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app movieservice --values=values.yml --namespace qa
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
                cp charts/cast-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app castservice --values=values.yml --namespace staging
                cp charts/movie-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app movieservice --values=values.yml --namespace staging
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
                cp charts/cast-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app castservice --values=values.yml --namespace prod
                cp charts/movie-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install app movieservice --values=values.yml --namespace prod
                '''
                }
            }

        }

}
}

