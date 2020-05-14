pipeline {

  environment {
    PROJECT = "demo"
  }

agent any
  }
  stages {
    stage('Deploy demo') {
      
      }
      steps {
        {
          withKubeConfig([credentialsId: '78a7be91-5339-4a89-88d4-515257366539']){
          // Create namespace if it doesn't exist
          sh("export KUBECONFIG=/var/lib/jenkins/test.kubeconfig")
          sh("/var/lib/jenkins/yandex-cloud/bin/yc container registry configure-docker")
          sh("/var/lib/jenkins/yandex-cloud/bin/yc managed-kubernetes cluster get-credentials --external --name k8s-demo --force")
          sh("/var/lib/jenkins/kubectl get ns deployment || kubectl create ns deployment")
          sh("/var/lib/jenkins/kubectl apply -n deployment -f /var/lib/jenkins/deployment.yml")
          sh("/var/lib/jenkins/kubectl get ns service || kubectl create ns service)
          sh("/var/lib/jenkins/kubectl apply -n service -f /var/lib/jenkins/service.yml")
          echo 'Done!'
          }
        }
      }
    }
  }
}
