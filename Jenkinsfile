pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE = 'samuelaka225/petshop'
    }

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    stages {
        stage('Clean workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/samuelaka12/DevSecOps-CI-CD-Pipeline.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn -B clean compile'
            }
        }

        stage('Unit tests') {
            steps {
                sh 'mvn -B test'
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectKey=petshop \
                          -Dsonar.projectName=Petshop \
                          -Dsonar.sources=src/main \
                          -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Package WAR') {
            steps {
                sh 'mvn -B package -DskipTests'
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                script {
                    def dcHome = tool 'dependency-check'
                    withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                        sh """
                            ${dcHome}/bin/dependency-check.sh \
                              --project petshop \
                              --scan target/*.war \
                              --format XML --format HTML \
                              --out . \
                              --data /var/lib/jenkins/dependency-check-data \
                              --nvdApiKey \$NVD_API_KEY
                        """
                    }
                }
                dependencyCheckPublisher pattern: 'dependency-check-report.xml'
            }
        }

        stage('Docker build') {
            steps {
                sh 'docker build -t $IMAGE:$BUILD_NUMBER -t $IMAGE:latest .'
            }
        }

        stage('Trivy image scan') {
            steps {
                sh '''
                    trivy image --no-progress \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      --format table \
                      --output trivy-report.txt \
                      $IMAGE:$BUILD_NUMBER
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DH_USER', passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        trap 'docker logout' EXIT
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$BUILD_NUMBER
                        docker push $IMAGE:latest
                    '''
                }
            }
        }

        stage('Deploy to k3s') {
            steps {
                withKubeConfig(credentialsId: 'k8s-petshop') {
                    sh '''
                        sed "s|IMAGE_TAG|$BUILD_NUMBER|" k8s/petshop.yaml | kubectl apply -f -
                        kubectl rollout status deployment/petshop -n petshop --timeout=300s
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.txt, dependency-check-report.html', allowEmptyArchive: true
            sh 'docker image prune -f'
        }
    }
}
