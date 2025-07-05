/* ──────────  CI/CD Moodle (estable) ────────── */

pipeline {
  agent any

  environment {
    REGISTRY        = 'docker.io'
    IMAGE_REPO      = 'bryanyaguarshungo/moodle'
    TAG             = "${env.GIT_COMMIT.take(7)}"
    DOCKER_CREDS_ID = 'docker-hub2'
    KUBECONFIG_ID   = 'kubeconfig'
    K8S_NAMESPACE   = 'default'
  }

  stages {

    /* 1 ─ Checkout */
    stage('Checkout') {
      steps { checkout scm }
    }

    /* 2 ─ Build & Push */
    stage('Build & Push Image') {
      steps {
        withCredentials([usernamePassword(
            credentialsId: DOCKER_CREDS_ID,
            usernameVariable: 'USER',
            passwordVariable: 'PASS')]) {

          sh """
            echo \$PASS | docker login -u \$USER --password-stdin $REGISTRY
            docker build -f app/Dockerfile -t $REGISTRY/$IMAGE_REPO:$TAG app
            docker push $REGISTRY/$IMAGE_REPO:$TAG
          """
        }
      }
    }

    /* 3 ─ Render & Deploy */
    stage('Render & Deploy to K8s') {
      steps {
        withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KCFG')]) {

          // Usamos kubectl oficial de Bitnami e instalamos kustomize en /usr/local/bin
          docker.image('bitnami/kubectl:1.30.0-debian-12-r0')
                .inside('--entrypoint=""') {

            sh '''
              set -e
              curl -sL https://github.com/kubernetes-sigs/kustomize/releases/download/v5.4.1/kustomize_v5.4.1_linux_amd64.tar.gz \
                | tar -xz -C /usr/local/bin

              cd "$WORKSPACE/infra"
              kustomize edit set image moodle=$REGISTRY/$IMAGE_REPO:$TAG
              kustomize build . | kubectl --kubeconfig="$KCFG" apply -n $K8S_NAMESPACE -f -
            '''
          }
        }
      }
    }

    /* 4 ─ Smoke Test */
    stage('Smoke Test') {
      steps {
        sh '''
          code=$(curl -s -o /dev/null -w '%{http_code}' \
                http://198.154.99.201:30080/login/index.php)
          [ "$code" = "200" ] && echo "✅ Smoke OK" || {
            echo "❌ Smoke FAIL ($code)"; exit 1; }
        '''
      }
    }
  }

  /* 5 ─ Rollback */
  post {
    failure {
      withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KCFG')]) {
        docker.image('bitnami/kubectl:1.30.0-debian-12-r0')
              .inside('--entrypoint=""') {
          sh 'kubectl --kubeconfig="$KCFG" rollout undo deployment/moodle -n $K8S_NAMESPACE || true'
        }
      }
    }
  }
}
