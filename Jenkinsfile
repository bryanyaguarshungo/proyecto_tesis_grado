/*  Jenkinsfile — CI/CD Moodle 100 % funcional  */

pipeline {
  /* ───────────── 1. El propio contenedor Jenkins será el agente ───────────── */
  agent any          // trae el CLI docker porque compartes /var/run/docker.sock

  /* ───────────── 2. Variables globales ───────────── */
  environment {
    REGISTRY        = 'docker.io'
    IMAGE_REPO      = 'bryanyaguarshungo/moodle'
    TAG             = "${env.GIT_COMMIT.take(7)}"
    DOCKER_CREDS_ID = 'dockerhub-creds'   // ← credencial tipo Usuario/Token en Jenkins
    KUBECONFIG_ID   = 'kubeconfig'        // ← credencial tipo File con permisos apply
    K8S_NAMESPACE   = 'default'
  }

  /* ───────────── 3. Pipeline ───────────── */
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

    /* Render + Deploy se ejecuta dentro de un contenedor que trae kubectl            */
    /* (y allí mismo instalamos kustomize).                                           */
    stage('Render & Deploy to K8s') {
      steps {
        withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KCFG')]) {

          docker.image('bitnami/kubectl:1.30').inside(
                  '-v /var/run/docker.sock:/var/run/docker.sock') {

            /* Instala kustomize solo la primera vez (2-3 s) */
            sh '''
              if ! command -v kustomize >/dev/null; then
                curl -s https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh | bash
                mv kustomize /usr/local/bin/
              fi
            '''

            /* Inyecta el tag recién construido y aplica */
            sh '''
              cd $WORKSPACE/infra
              kustomize edit set image moodle=${REGISTRY}/${IMAGE_REPO}:${TAG}
              kustomize build . > rendered.yaml
              kubectl --kubeconfig=$KCFG apply -f rendered.yaml -n ${K8S_NAMESPACE}
            '''
          }
        }
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          code=$(curl -s -o /dev/null -w '%{http_code}' http://198.154.99.201:30080/login/index.php)
          [ "$code" = "200" ] && echo "✅ Smoke OK" || { echo "❌ Smoke FAIL ($code)"; exit 1; }
        '''
      }
    }
  }   /* ────────── fin stages ────────── */

  /* ───────────── 4. Rollback si algo falla después del deploy ───────────── */
  post {
    failure {
      echo 'Rolling back…'
      withCredentials([file(credentialsId: KUBECONFIG_ID, variable: 'KCFG')]) {
        docker.image('bitnami/kubectl:1.30').inside {
          sh 'kubectl --kubeconfig=$KCFG rollout undo deployment/moodle -n ${K8S_NAMESPACE} || true'
        }
      }
    }
  }
}
