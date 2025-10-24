pipeline {
  agent any

  environment {
    GITHUB_USER     = "tinotenda-alfaneti"
    REPO_NAME       = "atarnet-uptimekuma"
    IMAGE_NAME      = "louislam/uptime-kuma"
    TAG             = "latest"
    APP_NAME        = "uptime-kuma"
    NAMESPACE       = "monitoring"
    KUBECONFIG_CRED = "kubeconfigglobal"
    PATH            = "$WORKSPACE/bin:$PATH"
  }

  stages {

    stage('Checkout Code') {
      steps {
        echo "📦 Checking out ${REPO_NAME}..."
        checkout scm
        sh 'mkdir -p $WORKSPACE/bin'
      }
    }

    stage('Install Tools') {
      steps {
        sh '''
          echo "⚙️ Installing kubectl & helm..."
          ARCH=$(uname -m)
          case "$ARCH" in
              x86_64)   KARCH=amd64 ;;
              aarch64)  KARCH=arm64 ;;
              armv7l)   KARCH=armv7 ;;
              *) echo "Unsupported arch: $ARCH" && exit 1 ;;
          esac

          VER=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
          curl -sLO https://storage.googleapis.com/kubernetes-release/release/${VER}/bin/linux/${KARCH}/kubectl
          chmod +x kubectl && mv kubectl $WORKSPACE/bin/

          HELM_VER="v3.14.4"
          curl -sLO https://get.helm.sh/helm-${HELM_VER}-linux-${KARCH}.tar.gz
          tar -zxf helm-${HELM_VER}-linux-${KARCH}.tar.gz
          mv linux-${KARCH}/helm $WORKSPACE/bin/helm
          chmod +x $WORKSPACE/bin/helm
          rm -rf linux-${KARCH} helm-${HELM_VER}-linux-${KARCH}.tar.gz
        '''
      }
    }

    stage('Verify Cluster Access') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            echo "🔐 Setting up kubeconfig..."
            mkdir -p $WORKSPACE/.kube
            cp "$KUBECONFIG_FILE" $WORKSPACE/.kube/config
            chmod 600 $WORKSPACE/.kube/config
            export KUBECONFIG=$WORKSPACE/.kube/config
            $WORKSPACE/bin/kubectl cluster-info
          '''
        }
      }
    }

    stage('Prepare Namespace & Secrets') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=$WORKSPACE/.kube/config
            echo "🧱 Ensuring namespace ${NAMESPACE} exists..."
            $WORKSPACE/bin/kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

            echo "✅ Namespace setup complete for ${NAMESPACE}"
          '''
        }
      }
    }


    stage('Trivy Scan') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=$WORKSPACE/.kube/config
            IMAGE_DEST="${IMAGE_NAME}:${TAG}"

            echo "🔍 Running Trivy scan for ${IMAGE_DEST}..."
            sed "s|__IMAGE_DEST__|${IMAGE_DEST}|g" $WORKSPACE/ci/kubernetes/trivy.yaml > trivy-job.yaml

            $WORKSPACE/bin/kubectl delete job trivy-scan -n ${NAMESPACE} --ignore-not-found=true
            $WORKSPACE/bin/kubectl apply -f trivy-job.yaml -n ${NAMESPACE}
            $WORKSPACE/bin/kubectl wait --for=condition=complete job/trivy-scan -n ${NAMESPACE} --timeout=5m || true
            $WORKSPACE/bin/kubectl logs job/trivy-scan -n ${NAMESPACE} || true
          '''
        }
      }
    }

    stage('Deploy with Helm') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=$WORKSPACE/.kube/config
            IMAGE_DEST="${IMAGE_NAME}:${TAG}"

            echo "⚙️ Deploying ${APP_NAME} with Helm..."
            $WORKSPACE/bin/helm upgrade --install ${APP_NAME} $WORKSPACE/charts/app \
              --namespace ${NAMESPACE} \
              --create-namespace \
              --set image=${IMAGE_DEST} \
              --wait --timeout 5m
          '''
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=$WORKSPACE/.kube/config
            echo "🩺 Running Helm test for ${APP_NAME}..."
            $WORKSPACE/bin/helm test ${APP_NAME} --namespace ${NAMESPACE} --logs --timeout 2m
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ ${APP_NAME} pipeline completed successfully."
    }
    failure {
      echo "❌ ${APP_NAME} pipeline failed."
    }
    always {
      echo "🧹 Cleaning up Kubernetes jobs..."
      withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
        sh '''
          export KUBECONFIG=$WORKSPACE/.kube/config
          kubectl delete job trivy-scan -n ${NAMESPACE} --ignore-not-found=true
          echo "Cleanup complete."
        '''
      }
      cleanWs()
    }
  }
}
