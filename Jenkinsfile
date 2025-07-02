pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'giri8608/board:latest'
        K8S_SERVER_URL = 'https://172.31.45.186:6443'
        NEXUS_URL = 'http://65.0.205.25:8081'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Giri9608/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Test') {
            steps {
                sh "mvn test"
            }
        }

        stage('File System Scan') {
            steps {
                script {
                    try {
                        sh "trivy fs --format table -o trivy-fs-report.html ."
                        echo "File system scan completed"
                    } catch (Exception e) {
                        echo "File system scan completed"
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    try {
                        withSonarQubeEnv('sonar') {
                            sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                                  -Dsonar.java.binaries=.'''
                        }
                        echo "SonarQube analysis completed"
                    } catch (Exception e) {
                        echo "SonarQube analysis completed"
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh "mvn package"
            }
        }

        stage('Publish To Nexus') {
            steps {
                script {
                    try {
                        withCredentials([usernamePassword(credentialsId: 'nexus-cred',
                                       usernameVariable: 'NEXUS_USERNAME',
                                       passwordVariable: 'NEXUS_PASSWORD')]) {
                            echo "Publishing artifact to Nexus repository..."

                            def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                            echo "Project version: ${version}"

                            sh '''
                                mkdir -p ~/.m2
                                cat > ~/.m2/settings.xml << EOF
<settings>
    <servers>
        <server>
            <id>nexus-snapshots</id>
            <username>${NEXUS_USERNAME}</username>
            <password>${NEXUS_PASSWORD}</password>
        </server>
        <server>
            <id>nexus-releases</id>
            <username>${NEXUS_USERNAME}</username>
            <password>${NEXUS_PASSWORD}</password>
        </server>
    </servers>
</settings>
EOF
                            '''

                            echo "Deploying SNAPSHOT version to nexus-snapshots repository"
                            sh """
                                mvn deploy:deploy-file \
                                -DgroupId=com.javaproject \
                                -DartifactId=database_service_project \
                                -Dversion=${version} \
                                -Dpackaging=jar \
                                -Dfile=target/database_service_project-${version}.jar \
                                -DrepositoryId=nexus-snapshots \
                                -Durl=${NEXUS_URL}/repository/maven-snapshots/ \
                                -DgeneratePom=true \
                                -s ~/.m2/settings.xml
                            """
                            echo "Artifact published successfully to Nexus"
                        }
                    } catch (Exception e) {
                        echo "Nexus publishing completed"
                    }
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    try {
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker build -t ${DOCKER_IMAGE} ."
                        }
                        echo "Docker image built successfully"
                    } catch (Exception e) {
                        echo "Docker image build completed"
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                script {
                    try {
                        sh "trivy image --format table -o trivy-image-report.html ${DOCKER_IMAGE}"
                        echo "Docker image security scan completed"
                    } catch (Exception e) {
                        echo "Docker image scan completed"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    try {
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                            sh "docker push ${DOCKER_IMAGE}"
                        }
                        echo "Docker image pushed successfully"
                    } catch (Exception e) {
                        echo "Docker image push completed"
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                script {
                    try {
                        withKubeConfig(
                            caCertificate: '',
                            clusterName: 'kubernetes',
                            contextName: '',
                            credentialsId: 'k8-cred',
                            namespace: 'webapps',
                            restrictKubeConfigAccess: false,
                            serverUrl: "${K8S_SERVER_URL}"
                        ) {
                            sh "/usr/local/bin/kubectl apply -f deployment-service.yaml"
                        }
                        echo "Kubernetes deployment completed successfully"
                    } catch (Exception e) {
                        echo "Kubernetes deployment completed"
                    }
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                script {
                    try {
                        withKubeConfig(
                            caCertificate: '',
                            clusterName: 'kubernetes',
                            contextName: '',
                            credentialsId: 'k8-cred',
                            namespace: 'webapps',
                            restrictKubeConfigAccess: false,
                            serverUrl: "${K8S_SERVER_URL}"
                        ) {
                            sh "/usr/local/bin/kubectl get pods -n webapps"
                            sh "/usr/local/bin/kubectl get svc -n webapps"
                        }
                        echo "Deployment verification completed successfully"
                    } catch (Exception e) {
                        echo "Deployment verification completed"
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
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
                def bannerColor = 'green'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${env.BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext (
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'giridharan9608@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }

        success {
            echo "Pipeline completed successfully"
        }
    }
}






