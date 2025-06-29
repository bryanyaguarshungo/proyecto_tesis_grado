/* ────────────────────────────────────────────────────────────────────
   Jenkinsfile definitivo – CI/CD Moodle
   Funciona con:
   • Jenkins/jenkins:lts  (socket Docker montado)
   • Plugin “Docker Pipeline”
   • Credenciales:
       - dockerhub-creds (username + token)
       - kubeconfig      (secret file)
   ──────────────────────────────────────────────────────────────────── */

pipeline {
  agent any   // usamos el propio contenedor Jenkins, ya tiene el CLI docker

  environment {
    REGISTRY        = 'docker.io'
    IMAGE_REPO      = 'bryanyaguarshungo/moodle'
    TAG             = "${env.GIT_COMMIT.take(7)}"
    DOCKER_CREDS_ID = 'github-token'
    KUBECONFIG_ID   = 'kubeconfig'
    K8S_NAMESPACE   = 'default'
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push Image') {
      steps {
        withCredentials([usernamePassword(
            credentialsId: DOCKER_CREDS_ID,
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

    /* Render + Deploy dentro de contenedor que trae kubectl  */
    stage('Render & Deploy to K8s') {
      steps {
        withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KCFG')]) {
          script {
            docker.image('bitnami/kubectl:1.30').inside(
                  '-v /var/run/docker.sock:/var/run/docker.sock') {

              /* instala kustomize si falta (2 s) */
              sh '''
                if ! command -v kustomize >/dev/null; then
                  curl -s https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh | bash
                  mv kustomize /usr/local/bin/
                fi
              '''

              /* renderiza y aplica */
              sh '''
                cd "$WORKSPACE/infra"
                kustomize edit set image moodle=${REGISTRY}/${IMAGE_REPO}:${TAG}
                kustomize build . > rendered.yaml
                kubectl --kubeconfig="$KCFG" apply -f rendered.yaml -n ${K8S_NAMESPACE}
              '''
            }
          }
        }
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          URL="http://198.154.99.201:30080/login/index.php"
          code=$(curl -s -o /dev/null -w '%{http_code}' "$URL")
          [ "$code" = "200" ] && echo "✅ Smoke OK" || {
            echo "❌ Smoke FAIL ($code)"; exit 1; }
        '''
      }
    }
  }

  post {
    failure {
      withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KCFG')]) {
        script {
          docker.image('bitnami/kubectl:1.30').inside {
            sh 'kubectl --kubeconfig="$KCFG" rollout undo deployment/moodle -n ${K8S_NAMESPACE} || true'
          }
        }
      }
    }
  }
}
