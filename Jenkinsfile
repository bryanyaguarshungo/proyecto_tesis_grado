pipeline {
  agent {
    docker {
      image 'bitnami/kubectl:1.30'
      args  '-v /var/run/docker.sock:/var/run/docker.sock'
    }
  }

  environment {
    REGISTRY        = 'docker.io'
    IMAGE_REPO      = 'bryanyaguarshungo/moodle'
    TAG             = "${env.GIT_COMMIT.take(7)}"
    DOCKER_CREDS_ID = 'dockerhub-creds'
    KUBECONFIG_ID   = 'kubeconfig'
    K8S_NAMESPACE   = 'default'
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: DOCKER_CREDS_ID,
                                          usernameVariable: 'USER',
                                          passwordVariable: 'PASS')]) {
          sh """
            docker login -u $USER -p $PASS $REGISTRY
            docker build -t $REGISTRY/$IMAGE_REPO:$TAG app
            docker push   $REGISTRY/$IMAGE_REPO:$TAG
          """
        }
      }
    }

    stage('Install kustomize') {
      steps {
        sh '''
          if ! command -v kustomize >/dev/null; then
            curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
            mv kustomize /usr/local/bin/
          fi
        '''
      }
    }

    stage('Render Manifests') {
      steps {
        dir('infra') {
          sh "kustomize edit set image moodle=$REGISTRY/$IMAGE_REPO:$TAG"
          sh "kustomize build . > rendered.yaml"
        }
      }
    }

    stage('Deploy to K8s') {
      steps {
        withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KCFG')]) {
          sh 'kubectl --kubeconfig=$KCFG apply -f infra/rendered.yaml -n $K8S_NAMESPACE'
        }
      }
    }

    stage('Smoke Test') {
      steps { sh 'app/tests/smoke.sh' }
    }
  }   // ←–––––– cierre del bloque stages

  post {
    failure {
      echo 'Rolling back…'
      sh 'kubectl rollout undo deployment/moodle -n $K8S_NAMESPACE || true'
    }
  }
}     // ←–––––– cierre final del bloque pipeline
