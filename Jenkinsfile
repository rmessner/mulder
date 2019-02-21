pipeline {
  agent {
    label "jenkins-go"
  }
  environment {
    ORG = 'rmessner'
    APP_NAME = 'mulder'
    CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
  }
  stages {
    stage('CI Build and push snapshot') {
      when {
        branch 'PR-*'
      }
      environment {
        PREVIEW_VERSION = "0.0.0-SNAPSHOT-$BRANCH_NAME-$BUILD_NUMBER"
        PREVIEW_NAMESPACE = "$APP_NAME-$BRANCH_NAME".toLowerCase()
        HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()
      }
      steps {
        container('go') {
          dir('/home/jenkins/go/src/github.com/rmessner/mulder') {
            checkout scm
            sh "make test-unit"
            sh "make linux"
            sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold.yaml"
            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          }
          dir('/home/jenkins/go/src/github.com/rmessner/mulder/charts/mulder') {
            sh "jx step helm build"
          } 
          dir('/home/jenkins/go/src/github.com/rmessner/mulder/charts/preview') {
            sh "make preview"
            sh "jx preview --app $APP_NAME --dir ../.."
          }
          dir('/home/jenkins/go/src/github.com/rmessner/mulder') {
            script {
              sleep 10
              addr=sh(script: "kubectl -n jx-$CHANGE_AUTHOR-$HELM_RELEASE get ing $APP_NAME -o jsonpath='{.spec.rules[0].host}'", returnStdout: true).trim()
              sh "MULDER_ADDR=$addr make test-integration"
            }
          }
        }
      }
    }
    stage('Build Release') {
      when {
        branch 'master'
      }
      steps {
        container('go') {
          dir('/home/jenkins/go/src/github.com/rmessner/mulder') {
            checkout scm

            // ensure we're not on a detached head
            sh "git checkout master"
            sh "git config --global credential.helper store"
            sh "jx step git credentials"

            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            sh "jx step tag --version \$(cat VERSION)"
            sh "make build"
            sh "export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml"
            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
          }
        }
      }
    }
    stage('Promote to Environments') {
      when {
        branch 'master'
      }
      steps {
        container('go') {
          dir('/home/jenkins/go/src/github.com/rmessner/mulder/charts/mulder') {
            sh "jx step changelog --version v\$(cat ../../VERSION)"

            // release the helm chart
            sh "jx step helm release"

            // promote through all 'Auto' promotion Environments
            sh "jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)"
          }
        }
      }
    }
  }
}
