pipeline {
  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    disableConcurrentBuilds()
  }
  agent any
  environment {
    IMAGE_NAME          = "nginx-proxy-manager"
    TEMP_IMAGE_NAME     = "nginx-proxy-manager-build_${BUILD_NUMBER}"
    TEMP_IMAGE_NAME_ARM = "nginx-proxy-manager-arm-build_${BUILD_NUMBER}"
    //TAG_VERSION         = getPackageVersion()
    TAG_VERSION         = "preview"
  }
  stages {
    stage('Prepare') {
      steps {
        sh 'docker pull $DOCKER_CI_TOOLS'
      }
    }
    stages {
      stage('Build') {
        parallel {
          stage('x86_64') {
            steps {
              ansiColor('xterm') {
                // Codebase
                sh 'docker run --rm -v $(pwd):/srv/app -w /srv/app $IMAGE_NAME-base:latest yarn --registry=$NPM_REGISTRY install'
                sh 'docker run --rm -v $(pwd):/srv/app -w /srv/app $IMAGE_NAME-base:latest npm runscript build'
                sh 'rm -rf node_modules'
                sh 'docker run --rm -v $(pwd):/srv/app -w /srv/app $IMAGE_NAME-base:latest yarn --registry=$NPM_REGISTRY install --prod'
                sh 'docker run --rm -v $(pwd):/data $DOCKER_CI_TOOLS node-prune'

                // Docker Build
                sh 'docker build --pull --no-cache --squash --compress -t $TEMP_IMAGE_NAME .'

                // Private Registry
                sh 'docker tag $TEMP_IMAGE_NAME $DOCKER_PRIVATE_REGISTRY/$IMAGE_NAME:$TAG_VERSION'
                sh 'docker push $DOCKER_PRIVATE_REGISTRY/$IMAGE_NAME:$TAG_VERSION'

                // Dockerhub
                sh 'docker tag $TEMP_IMAGE_NAME docker.io/jc21/$IMAGE_NAME:$TAG_VERSION'

                withCredentials([usernamePassword(credentialsId: 'jc21-dockerhub', passwordVariable: 'dpass', usernameVariable: 'duser')]) {
                  sh "docker login -u '${duser}' -p '$dpass'"
                  sh 'docker push docker.io/jc21/$IMAGE_NAME:$TAG_VERSION'
                }

                sh 'docker rmi $TEMP_IMAGE_NAME'
              }
            }
          }
          stage('armhf') {
            agent {
              label 'armhf'
            }
            steps {
              ansiColor('xterm') {
                // Codebase
                sh 'docker run --rm -v $(pwd):/srv/app -w /srv/app $IMAGE_NAME-base:armhf yarn --registry=$NPM_REGISTRY install'
                sh 'docker run --rm -v $(pwd):/srv/app -w /srv/app $IMAGE_NAME-base:armhf npm runscript build'
                sh 'rm -rf node_modules'
                sh 'docker run --rm -v $(pwd):/srv/app -w /srv/app $IMAGE_NAME-base:armhf yarn --registry=$NPM_REGISTRY install --prod'

                // Docker Build
                 sh 'docker build --pull --no-cache --squash --compress -t TEMP_IMAGE_NAME_ARM .'

                // Private Registry
                sh 'docker tag TEMP_IMAGE_NAME_ARM $DOCKER_PRIVATE_REGISTRY/$IMAGE_NAME:$TAG_VERSION-armhf'
                sh 'docker push $DOCKER_PRIVATE_REGISTRY/$IMAGE_NAME:$TAG_VERSION-armhf'

                // Dockerhub
                sh 'docker tag TEMP_IMAGE_NAME_ARM docker.io/jc21/$IMAGE_NAME:$TAG_VERSION-armhf'

                withCredentials([usernamePassword(credentialsId: 'jc21-dockerhub', passwordVariable: 'dpass', usernameVariable: 'duser')]) {
                  sh "docker login -u '${duser}' -p '$dpass'"
                  sh 'docker push docker.io/jc21/$IMAGE_NAME:$TAG_VERSION-armhf'
                }

                sh 'docker rmi TEMP_IMAGE_NAME_ARM'
              }
            }
          }
        }
      }
    }
  }
  post {
    success {
      juxtapose event: 'success'
      sh 'figlet "SUCCESS"'
    }
    failure {
      juxtapose event: 'failure'
      sh 'figlet "FAILURE"'
    }
  }
}

def getPackageVersion() {
  ver = sh(script: 'docker run --rm -v $(pwd):/data $DOCKER_CI_TOOLS bash -c "cat /data/package.json|jq -r \'.version\'"', returnStdout: true)
  return ver.trim()
}