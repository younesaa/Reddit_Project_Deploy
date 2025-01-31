pipeline{
    agent any
    tools{
        jdk 'jdk'
        nodejs 'nodejs'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/uniquesreedhar/Reddit-Project.git'
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Reddit \
                    -Dsonar.projectKey=Reddit '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker build -t reddit ."
                       sh "docker tag reddit sreedhar8897/reddit:${BUILD_NUMBER} "
                       sh "docker push sreedhar8897/reddit:${BUILD_NUMBER} "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image sreedhar8897/reddit:latest > trivy.txt"
            }
        }
        stage('Update Deployment file') {
            environment {
                GIT_REPO_NAME = "Reddit-Project"
                GIT_USER_NAME = "uniquesreedhar"
            }
            steps {
                dir('K8s'){
                    withCredentials([string(credentialsId: 'githubcred', variable: 'GITHUB_TOKEN')]) {
                        sh '''
                            git config user.email "madithatisreedhar123@gmail.com"
                            git config user.name "uniquesreedhar"
                            BUILD_NUMBER=${BUILD_NUMBER}
                            echo $BUILD_NUMBER
                            imageTag=$(grep -oP '(?<=reddit:)[^ ]+' deployment.yml)
                            echo $imageTag
                            sed -i "s/reddit:${imageTag}/reddit:${BUILD_NUMBER}/" deployment.yml
                            git add deployment.yml
                            git commit -m "Update deployment Image to version \${BUILD_NUMBER}"
                            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                        '''
                    }
                }
            }
        }
    }
}
