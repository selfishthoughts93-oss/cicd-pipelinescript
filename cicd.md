node {

    // 1. GitHub Code Checkout
    stage('GitHub Code Checkout') {

        git credentialsId: 'rushi-github-credentials',
            url: 'https://github.com/devopsbyrushi/rushiDevOpsTraining-FlipkartWarDemo.git'
    }


    // 2. Maven Build
    stage('Maven Build') {

        sh 'mvn clean install'
    }


    // 3. Verify WAR File
    stage('Verify WAR File') {

        sh 'ls -lh target/flipkart-clone.war'
    }


    // 4. SonarQube Code Analysis
    stage('SonarQube Analysis') {

        withSonarQubeEnv('SonarQube') {

            sh '''
                mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar \
                -Dsonar.projectKey=flipkart-project \
                -Dsonar.projectName=Flipkart-Application
            '''
        }
    }


    // 5. SonarQube Quality Gate
    stage('SonarQube Quality Gate') {

        timeout(time: 5, unit: 'MINUTES') {

            def qualityGate = waitForQualityGate()

            if (qualityGate.status != 'OK') {

                error "SonarQube Quality Gate Failed: ${qualityGate.status}"
            }
        }
    }


    // 6. Docker Image Build
    stage('Docker Build') {

        sh '''
            docker build -t devopsbyrushi/octsecurebankcicdpipleline:v1 .
        '''
    }


    // 7. Docker Hub Login
    stage('Docker Login') {

        withCredentials([usernamePassword(
            credentialsId: 'dockerhub-passwd',
            usernameVariable: 'DOCKER_USER',
            passwordVariable: 'DOCKER_PASS'
        )]) {

            sh '''
                echo "$DOCKER_PASS" | docker login \
                -u "$DOCKER_USER" --password-stdin
            '''
        }
    }


    // 8. Push Docker Image to Docker Hub
    stage('Docker Hub Push') {

        sh '''
            docker push devopsbyrushi/octsecurebankcicdpipleline:v1
        '''
    }


    // 9. Connect Jenkins to GKE
    stage('Connect to GKE') {

        withCredentials([file(
            credentialsId: 'GKE-SA',
            variable: 'GKE_SA'
        )]) {

            sh '''
                gcloud auth activate-service-account \
                    --key-file="$GKE_SA" \
                    --project=august2027

                gcloud config set project august2027

                gcloud container clusters get-credentials rushi-gke \
                    --zone=us-central1-f \
                    --project=august2027

                kubectl get nodes
            '''
        }
    }


    // 10. Deploy Application to GKE
    stage('Deploy to GKE') {

        sh '''
            kubectl apply -f k8s/

            kubectl rollout restart deployment/flipkart-app

            kubectl rollout status deployment/flipkart-app \
                --timeout=300s
        '''
    }


    // 11. Verify Deployment
    stage('Verify Deployment') {

        sh '''
            kubectl get deployments

            kubectl get pods -o wide

            kubectl get services
        '''
    }


    // 12. Pipeline Completed
    stage('Pipeline Completed') {

        echo 'Flipkart CI/CD Pipeline Completed Successfully!'
    }

}
