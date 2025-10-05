pipeline {
  agent any

  environment {
    // Jenkins VM IP for SonarQube access (35.224.107.186)
    SONARQUBE = 'sonarqube'
    SCANNER = 'SonarScanner'
    SONAR_PROJECT_KEY = 'hello-python'
    SONAR_API_TOKEN = credentials('sonar-token') 
    SONAR_HOST_URL = 'http://35.224.107.186:9000' 
  }

  stages {
    stage('Checkout') {
      steps {
        // Your GitHub repository URL (ishitahq)
        git branch: 'main', url: 'https://github.com/ishitahq/hello-python.git'
      }
    }

    stage('Install & Run Tests') {
      steps {
        sh '''
          echo "Installing dependencies..."
          python3 -m pip install --upgrade pip
          pip3 install --user -r requirements.txt
          
          echo "Running tests..."
          # FIX: Explicitly set PYTHONPATH to current directory ($PWD) 
          # This resolves the "No module named 'app'" error.
          env PYTHONPATH=$PWD python3 -m pytest -q
        '''
      }
    }

    stage('SonarQube Analysis (Async)') {
      steps {
        withSonarQubeEnv("${SONARQUBE}") {
          withEnv(["PATH+SONAR=${tool name: SCANNER}/bin"]) {
            sh '''
              sonar-scanner \
                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                -Dsonar.sources=. \
                -Dsonar.host.url=${SONAR_HOST_URL} \
                -Dsonar.login=${SONAR_API_TOKEN} \
                -Dsonar.python.version=3.10
            '''
          }
        }
      }
    }

    stage('Deploy to App VM') {
      steps {
        // Uses the SSH key stored in Jenkins credentials with ID 'gce-ssh'
        sshagent(credentials: ['gce-ssh']) {
          sh '''
            echo "Deploying to App VM..."
            
            # App VM User (ishu4cloud) and IP (35.184.238.185) used for deployment:
            
            # 1. Create directory if it doesn't exist
            ssh -o StrictHostKeyChecking=no ishu4cloud@35.184.238.185 "mkdir -p /home/ishu4cloud/app"
            
            # 2. Copy all files/folders to the App VM
            scp -o StrictHostKeyChecking=no -r * ishu4cloud@35.184.238.185:/home/ishu4cloud/app/
            
            # 3. Reload systemd, restart the Flask app service, and ensure it's enabled
            ssh -o StrictHostKeyChecking=no ishu4cloud@35.184.238.185 "sudo systemctl daemon-reload && sudo systemctl restart flaskapp && sudo systemctl enable flaskapp"
            
            echo "Deployment complete. You can curl http://35.184.238.185:8080"
          '''
        }
      }
    }

    stage('Optional: Check SonarQube Quality Gate') {
      steps {
        script {
          echo "Fetching SonarQube Quality Gate result (non-blocking)..."
          sh """
            curl -s -u ${SONAR_API_TOKEN}: '${SONAR_HOST_URL}/api/qualitygates/project_status?projectKey=${SONAR_PROJECT_KEY}' \
            | jq '.projectStatus.status'
          """
          echo "Inspect the SonarQube dashboard for full details."
        }
      }
    }
  }

  post {
    success { echo "Pipeline Succeeded" }
    failure { echo "Pipeline Failed" }
  }
}
