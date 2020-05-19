pipeline {

  environment {
    PROJECT = "demo"
  }
  agent any
  
  
  stages {
    stage('Deploy demo') {
      // Developer Branches
      steps {

          // Create namespace if it doesn't exist
          sh("/kubectl get ns master || kubectl create ns master")
          sh("kubectl apply -n master -f /var/lib/jenkins/deployment.yml")
          sh("kubectl apply -n master -f /var/lib/jenkins/service.yml")
          echo 'Done!'
          
        }
    }
  }
}
