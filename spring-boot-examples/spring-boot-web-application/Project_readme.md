"Setting up CI/CD Pipelines for Docker Kubernetes Project (hosted on Google Cloud Platform)"

A sample project is hosted here on Google Cloud Platform Kubernetes Engine.

We will set up an env_vars/application.properties file for our new project. 
Then we set up the multibranch pipeline in Jenkins.

Understanding the Code
env_vars

In order to customize this pipeline for our project, I updated the env_vars/application.properties file according to our project.

APP_NAME 	The application name - this will be used to create image name in Jenkins file 	spring
IMAGE_NAME 	This is the image name we want our project to be published at docker registry. 
PROJECT_NAME  	krupakar

DOCKER_REGISTRY_URL
	URL of the Docker registry.	registry.hub.docker.com

RELEASE_TAG
	Release tag for Docker image.	1.0.0

DOCKER_PROJECT_NAMESPACE
	Docker project namespace. My account on Docker Hub is krupakar which is also my default namespace

JENKINS_DOCKER_CREDENTIALS_ID
	This is the username password credential which will be added to Jenkins for login to Docker registry.
  (If you are using Openshift, you may want to login with  $(oc whoami -t)  for token 	

JENKINS_DOCKER_CREDENTIALS_ID

JENKINS_GCLOUD_CRED_ID
	This is the Google Cloud Platform service account key which is added to Jenkins as a file credential.
JENKINS_GCLOUD_CRED_ID

JENKINS_GCLOUD_CRED_LOCATION
	Unused. (If you prefer to not add file credential to Jenkins and to store the service account key at Jenkins and directly access from slave then use this) 	/var/lib/jenkins/lateral-ceiling-220011-5c9f0bd7782f.json

GCLOUD_PROJECT_ID
	This is the Google Cloud Project ID 	lateral-ceiling-220011

GCLOUD_K8S_CLUSTER_NAME
	This is our cluster name on Google Cloud 	


Dockerfile

 Here is a brief description of what this Dockerfile is doing:

    We build the image on top of maven:3.3.9-jdk-8 image
    We take in some of the overridable arguments and set environment parameters from them
    We create the $APP_HOME_DIR   directory and a user krupakar
    We copy the source into the context in $APP_BUILD_DIR   directory,
    In APP_BUILD_DIR   we build the code and move the built JAR to $APP_HOME_DIR   directory
    We set the permission and run the application with krupakar, the user we created earlier
    Entrypoint is an overridable simple pass-through script which calls the run command.
    The run script is created during the build with appropriate parameters like SPRING_PROFILES_ACTIVE, API_FULL_NAME, etc.

FROM maven:3.3.9-jdk-8

ARG RELEASE_VERSION=1.0.0-SNAPSHOT

ARG API_NAME=blogpost-api

ARG API_BUILD_DIR=/opt/usr/src

ARG APP_HOME_DIR=/var/www/app

ARG SPRING_PROFILES_ACTIVE=dev

ENV RELEASE_VERSION ${RELEASE_VERSION}

ENV API_FULL_NAME ${API_NAME}-${RELEASE_VERSION}

ENV API_BUILD_DIR ${API_BUILD_DIR}

ENV APP_HOME_DIR ${APP_HOME_DIR}

ENV SPRING_PROFILES_ACTIVE ${SPRING_PROFILES_ACTIVE}

EXPOSE 8080 8081

USER root

RUN mkdir -p ${APP_HOME_DIR} \

    && groupadd -g 10000 krupakar \

    && useradd --home-dir ${APP_HOME_DIR} -u 10000 -g krupakar krupakar

COPY . ${API_BUILD_DIR}

RUN cd ${API_BUILD_DIR}/ \

    && mvn clean package -Pjar -Dapi_name=${API_NAME} -Drelease_version=${RELEASE_VERSION} \

    && cp ${API_BUILD_DIR}/target/${API_FULL_NAME}.jar ${APP_HOME_DIR}/ \

    && cp ${API_BUILD_DIR}/files/entrypoint ${APP_HOME_DIR}/ \

    && echo "java -jar -Dspring.profiles.active=${SPRING_PROFILES_ACTIVE} ${APP_HOME_DIR}/${API_FULL_NAME}.jar" > ${APP_HOME_DIR}/run \

    && chmod -R 0766 ${APP_HOME_DIR} \

    && chown -R krupakar:krupakar ${APP_HOME_DIR} \

    && chmod g+w /etc/passwd

WORKDIR ${APP_HOME_DIR}

USER krupakar

ENTRYPOINT [ "./entrypoint" ]

CMD ["./run"]


Kubernetes Deployment, Service and Ingress


Deployment.yml

apiVersion: extensions/v1beta1

kind: Deployment

metadata:

  name: spring

spec:

  replicas: 1

  template:

    metadata:

      labels:

        app: spring

    spec:

      containers:

      - name: spring

        image: Sample_java_docker_image

        volumeMounts:

        - name: tls-secrets

          readOnly: true

          mountPath: "/var/www/app/tls"

        ports:

        - containerPort: 8080

          name: http

        imagePullPolicy: Always

      volumes:

      - name: tls-secrets

        secret:

          secretName: api-tls-secret


Service.yml

apiVersion: v1

kind: Service

metadata:

  labels:

    app: spring

  name: spring-svc

spec:

  ports:

  - port: 8080

    targetPort: 8080

    name: http

  selector:

    app: spring

  sessionAffinity: None

  type: NodePort


Ingress

Our ingress exposes the http-port of our service outside the cluster.
Our ingress utilizes the secret mentioned below to import two values. 
The values in our secret is are a self-signed secret and we are not using it as of now to keep the things simple. However, the code can be used to understand the use of secrets:

apiVersion: extensions/v1beta1

kind: Ingress

metadata:

  name: spring-ingress

  annotations:

    kubernetes.io/ingress.allow-http: "true"

spec:

  tls:

  - secretName: api-tls-secret

  backend:

    serviceName: spring-svc

    servicePort: http


Secret

Here are the contents of the secret file:

apiVersion: v1

data:

  tls.crt: "<<---base64 encoded ssl cert--->>"

  tls.key: "<<---base64 encoded cert key--->>"

kind: Secret

metadata:

  name: api-tls-secret

  namespace: default

type: Opaque


Jenkins Pipeline
Initialization

In our initialization stage, we basically take most of the parameters from the env_vars/application.properties files as described above. The timestamp is taken from the wrapper script below:

def getTimeStamp(){

    return sh (script: "date +'%Y%m%d%H%M%S%N' | sed 's/[0-9][0-9][0-9][0-9][0-9][0-9]\$//g'", returnStdout: true);

}


And the following function reads the values from env_vars/application.properties file:

def getEnvVar(String paramName){

    return sh (script: "grep '${paramName}' env_vars/project.properties|cut -d'=' -f2", returnStdout: true).trim();

}


Here's our initialization stage:

    stage('Init'){

        steps{

            //checkout scm;

        script{

        env.BASE_DIR = pwd()

        env.CURRENT_BRANCH = env.BRANCH_NAME

        env.IMAGE_TAG = getImageTag(env.CURRENT_BRANCH)

        env.TIMESTAMP = getTimeStamp();

        env.APP_NAME= getEnvVar('APP_NAME')

        env.IMAGE_NAME = getEnvVar('IMAGE_NAME')

        env.PROJECT_NAME=getEnvVar('PROJECT_NAME')

        env.DOCKER_REGISTRY_URL=getEnvVar('DOCKER_REGISTRY_URL')

        env.RELEASE_TAG = getEnvVar('RELEASE_TAG')

        env.DOCKER_PROJECT_NAMESPACE = getEnvVar('DOCKER_PROJECT_NAMESPACE')

        env.DOCKER_IMAGE_TAG= "${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${APP_NAME}:${RELEASE_TAG}"

        env.JENKINS_DOCKER_CREDENTIALS_ID = getEnvVar('JENKINS_DOCKER_CREDENTIALS_ID')        

        env.JENKINS_GCLOUD_CRED_ID = getEnvVar('JENKINS_GCLOUD_CRED_ID')

        env.GCLOUD_PROJECT_ID = getEnvVar('GCLOUD_PROJECT_ID')

        env.GCLOUD_K8S_CLUSTER_NAME = getEnvVar('GCLOUD_K8S_CLUSTER_NAME')

        env.JENKINS_GCLOUD_CRED_LOCATION = getEnvVar('JENKINS_GCLOUD_CRED_LOCATION')

        }

        }

    }


Cleanup

Our cleanup script simply clears our any dangling or stale images.

    stage('Cleanup'){

        steps{

            sh '''

            docker rmi $(docker images -f 'dangling=true' -q) || true

            docker rmi $(docker images | sed 1,2d | awk '{print $3}') || true

            '''

        }

    }


Build

Here we docker build   our project. Please notice that since we will be pushing our image to Dockerhub, the tag we are using contains DOCKER_REGISTRY_URL   
which is registry.hub.docker.com and my DOCKER_PROJECT_NAMESPACE   
    stage('Build'){

        steps{

            withEnv(["APP_NAME=${APP_NAME}", "PROJECT_NAME=${PROJECT_NAME}"]){

                sh '''

                docker build -t ${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG} --build-arg APP_NAME=${IMAGE_NAME}  -f app/Dockerfile app/.

                '''

            }   

        }

    }


Publish

In order to publish our image to the Docker registry, we make use of Jenkins credentials defined with variable JENKINS_DOCKER_CREDENTIALS_ID. 

    stage('Publish'){

        steps{

            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${JENKINS_DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWD']])

            {

            sh '''

            echo $DOCKER_PASSWD | docker login --username ${DOCKER_USERNAME} --password-stdin ${DOCKER_REGISTRY_URL} 

            docker push ${DOCKER_REGISTRY_URL}/${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}

            docker logout

            '''

            }

        }

    }


Deploy

In our Deploy stage, we make use of Jenkins secret file credential set up in the JENKINS_GCLOUD_CRED_ID   variable.
Again, to check how this variable is set up, please refer to first article.

For deployment, we process our deployment, service, and ingress files mentioned above using our simple script named process_files.sh. 
#!/bin/bash

if (($# <5))

  then

    echo "Usage : $0 <DOCKER_PROJECT_NAME> <APP_NAME> <IMAGE_TAG> <directory containing k8s files> <timestamp>"

    exit 1

fi

PROJECT_NAME=$1

APP_NAME=$2

IMAGE=$3

WORK_DIR=$4

TIMESTAMP=$5

main(){

find $WORK_DIR -name *.yml -type f -exec sed -i.bak1 's#__PROJECT_NAME__#'$PROJECT_NAME'#' {} \;

find $WORK_DIR -name *.yml -type f -exec sed -i.bak2 's#__APP_NAME__#'$APP_NAME'#' {} \;

find $WORK_DIR -name *.yml -type f -exec sed -i.bak3  's#__IMAGE__#'$IMAGE'#' {} \;

find $WORK_DIR -name *.yml -type f -exec sed -i.bak3  's#__TIMESTAMP__#'$TIMESTAMP'#' {} \;

}

main


And here is our Deployment stage. We activate our gcloud credential, 
we process our templates using the process_files.sh script mentioned above, 
then we use kubectl to apply our processed templates. 
We watch our rollout using  kubectl rollout status  command:

    stage('Deploy'){

        steps{

        withCredentials([file(credentialsId: "${JENKINS_GCLOUD_CRED_ID}", variable: 'JENKINSGCLOUDCREDENTIAL')])

        {

        sh """

            gcloud auth activate-service-account --key-file=${JENKINSGCLOUDCREDENTIAL}

            gcloud config set compute/zone asia-southeast1-a

            gcloud config set compute/region asia-southeast1

            gcloud config set project ${GCLOUD_PROJECT_ID}

            gcloud container clusters get-credentials ${GCLOUD_K8S_CLUSTER_NAME}

            chmod +x $BASE_DIR/k8s/process_files.sh

            cd $BASE_DIR/k8s/

            ./process_files.sh "$GCLOUD_PROJECT_ID" "${IMAGE_NAME}" "${DOCKER_PROJECT_NAMESPACE}/${IMAGE_NAME}:${RELEASE_TAG}" "./${IMAGE_NAME}/" ${TIMESTAMP}

            cd $BASE_DIR/k8s/${IMAGE_NAME}/.

            kubectl apply --force=true --all=true --record=true -f $BASE_DIR/k8s/$IMAGE_NAME/

            kubectl rollout status --watch=true --v=8 -f $BASE_DIR/k8s/$IMAGE_NAME/$IMAGE_NAME-deployment.yml

            gcloud auth revoke --all

            """

        }

        }

    }
