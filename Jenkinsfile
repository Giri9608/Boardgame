pipeline {
    agent any

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = 'giri8608/board:latest'
        K8S_SERVER_URL = 'https://172.31.46.58:6443'
        NEXUS_URL = 'http://13.201.60.194:8081'
    }

    stages {
        stage('Git Checkout') {
            steps {
                script {
                    // Clean workspace and checkout
                    deleteDir()
                    git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Giri9608/Boardgame.git'

                    // Verify files are present
                    sh "echo 'Workspace contents:'"
                    sh "ls -la"
                    sh "echo 'YAML files:'"
                    sh "ls -la *.yaml || echo 'No YAML files found'"
                }
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
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                          -Dsonar.java.binaries=.'''
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
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t ${DOCKER_IMAGE} ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html ${DOCKER_IMAGE}"
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push ${DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                script {
                    // Ensure deployment file exists
                    if (!fileExists('deployment-service.yaml')) {
                        echo "deployment-service.yaml not found, downloading from repository..."
                        sh "curl -o deployment-service.yaml https://raw.githubusercontent.com/Giri9608/Boardgame/main/deployment-service.yaml"
                    }

                    // Verify file and show contents
                    sh "echo 'Deployment file contents:'"
                    sh "cat deployment-service.yaml"
                }

                withKubeConfig(
                    caCertificate: '',
                    clusterName: 'kubernetes',
                    contextName: '',
                    credentialsId: 'k8-cred',
                    namespace: 'webapps',
                    restrictKubeConfigAccess: false,
                    serverUrl: "${K8S_SERVER_URL}"
                ) {
                    sh "/usr/local/bin/kubectl apply -f deployment-service.yaml --validate=false"
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
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
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'SUCCESS'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

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
    }
}



