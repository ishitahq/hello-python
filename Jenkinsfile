pipeline {
  agent any

  environment {
    SONARQUBE = 'sonarqube'
    SCANNER = 'SonarScanner'
    SONAR_PROJECT_KEY = 'hello-python'
    SONAR_API_TOKEN = credentials('sonar-token') 
    // CHANGE 1: Updated Jenkins VM IP for SonarQube host URL
    SONAR_HOST_URL = 'http://35.224.107.186:9000'
  }

  stages {
    stage('Checkout') {
      steps {
        // CHANGE 2: Updated GitHub Username
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
          python3 -m pytest -q
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
        sshagent(credentials: ['gce-ssh']) {
          sh '''
            echo "Deploying to App VM..."
            
            # CHANGE 3: Update App VM User (ishu4cloud) and IP (35.184.238.185) for mkdir
            ssh -o StrictHostKeyChecking=no ishu4cloud@35.184.238.185 "mkdir -p /home/ishu4cloud/app"
            
            # CHANGE 4: Update App VM User (ishu4cloud) and IP (35.184.238.185) for scp
            scp -o StrictHostKeyChecking=no -r * ishu4cloud@35.184.238.185:/home/ishu4cloud/app/
            
            # CHANGE 5: Update App VM User (ishu4cloud) and IP (35.184.238.185) for systemctl restart
            ssh -o StrictHostKeyChecking=no ishu4cloud@35.184.238.185 "sudo systemctl daemon-reload && sudo systemctl restart flaskapp && sudo systemctl enable flaskapp"
            
            # CHANGE 6: Update App VM IP (35.184.238.185) for verification echo
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

  stage('Install & Run Tests') {
      steps {
        sh '''
          echo "Installing dependencies..."
          python3 -m pip install --upgrade pip
          pip3 install --user -r requirements.txt
          
          echo "Running tests..."
          # STRONGER FIX: Set PYTHONPATH to current directory ($PWD) for this session
          # This ensures 'app' can be imported by the test runner.
          PYTHONPATH=$PWD python3 -m pytest -q
        '''
      }
  }

  post {
    success { echo "Pipeline Succeeded" }
    failure { echo "Pipeline Failed" }
  }
}
