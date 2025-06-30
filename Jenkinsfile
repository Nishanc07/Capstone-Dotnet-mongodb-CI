pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-token', url: 'https://github.com/Nishanc07/Capstone-Dotnet-mongodb-CI.git'
            }
        }

        stage('GitLeaks Scan') {
            steps {
                sh '''
                    gitleaks detect --report-format json --report-path gitleaks-report.json --exit-code 1
                '''
            }
        }

        stage('Compile') {
            steps {
                sh 'dotnet build'
            }
        }

        stage('Trivy Fs Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('Unit Testing') {
            steps {
                sh 'dotnet test'
            }
        }

        stage('Sonarqube analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Noteapp \
                        -Dsonar.projectKey=Noteapp \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token1'
                }
            }
        }

        stage('Build Image to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t nishanc7/noteapp:${IMAGE_TAG} ."
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html nishanc7/noteapp:${IMAGE_TAG}"
            }
        }
      stage('Push Image to DockerHub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push nishanc7/noteapp:${IMAGE_TAG}"
                    }
                }
            }
        }


        stage('Update Manifest File CD Repo') {
            steps {
                script {
                    cleanWs()
                    withCredentials([usernamePassword(credentialsId: 'git-token', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Nishanc07/Capstone-Dotnet-manodb-CD.git
                            cd Capstone-Dotnet-manodb-CD
                            sed -i 's|nishanc7/noteapp:.*|nishanc7/noteapp:${IMAGE_TAG}|' Manifest/manifest.yaml
                            echo 'updated manifest file content'
                            cat Manifest/manifest.yaml
                            git config user.name "jenkins"
                            git config user.email "jenkins@example.com"
                            git add Manifest/manifest.yaml
                            git commit -m "update image tag to ${IMAGE_TAG}"
                            git push origin main
                        """
                    }
                }
            }
        }

        
    }
      post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'nishanarayan604@gmail.com',
                    from: 'nishanc604@gmail.com',
                    replyTo: 'jenkins@nisha.com',
                    mimeType: 'text/html'
                )
            }
        }
    }
}
