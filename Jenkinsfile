pipeline {
    agent {
      kubernetes  {
            label 'jenkins-slave'
             defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:latest
    command:
    - cat
    tty: true
    volumeMounts:
    - name: dockersock
      mountPath: /var/run/docker.sock
  - name: tools
    image: argoproj/argo-cd-ci-builder:v1.0.0
    command:
    - cat
    tty: true
  volumes:
  - name: dockersock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }
  environment {
      IMAGE_REPO = "quay.io/rh_ee_lfalero/rsvp" // CAMBIAR
  }
  stages {
    stage('Build') {
      environment {
        DOCKERHUB_CREDS = credentials('quayio')
      }
      steps {
        container('docker') {
          sh "echo ${env.GIT_COMMIT}"
          // Build new image
          sh "until docker container ls; do sleep 3; done && docker image build -t ${env.IMAGE_REPO}:${env.GIT_COMMIT} ."
          // Publish new image
          sh "docker login --username $DOCKERHUB_CREDS_USR --password $DOCKERHUB_CREDS_PSW quay.io && docker image push ${env.IMAGE_REPO}:${env.GIT_COMMIT}"
        }
      }
    }
    stage('Deploy') {
      environment {
        GIT_CREDS = credentials('github')
        HELM_GIT_REPO_URL = "github.com/luisfalero/rsvpapp-helm-cicd.git" // CAMBIAR
        GIT_REPO_EMAIL = 'lfalero@redhat.com' // CAMBIAR
        GIT_REPO_BRANCH = "master"
          
       // Update above variables with your user details
      }
      steps {
        container('tools') {
            sh "git clone https://${env.HELM_GIT_REPO_URL}"
            sh "git config --global user.email ${env.GIT_REPO_EMAIL}"
             // install wq
            sh "wget https://github.com/mikefarah/yq/releases/download/v4.9.6/yq_linux_amd64.tar.gz"
            sh "tar xvf yq_linux_amd64.tar.gz"
            sh "mv yq_linux_amd64 /usr/bin/yq"
            sh "git checkout -b master"
          dir("rsvpapp-helm-cicd") {
              sh "git checkout ${env.GIT_REPO_BRANCH}"
            //install done
            sh '''#!/bin/bash
              echo $GIT_REPO_EMAIL
              echo $GIT_COMMIT
              ls -lth
              yq eval '.image.repository = env(IMAGE_REPO)' -i values.yaml
              yq eval '.image.tag = env(GIT_COMMIT)' -i values.yaml
              cat values.yaml
              pwd
              git add values.yaml
              git commit -m 'Triggered Build'
              git push https://$GIT_CREDS_USR:$GIT_CREDS_PSW@github.com/$GIT_CREDS_USR/rsvpapp-helm-cicd.git
            '''
          }
        }
      }
    }   
  }
}
