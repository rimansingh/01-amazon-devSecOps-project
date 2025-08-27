pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/rimansingh/01-amazon-devSecOps-project.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=amazon \
                        -Dsonar.projectKey=amazon '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    timeout(time: 3, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                    }
                }
            }
        }

        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }
        
        stage("OWASP FS Scan") {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan ./ 
                    --disableYarnAudit 
                    --disableNodeAudit 
                
                   ''',
                odcInstallation: 'dp-check'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Build Docker Image") {
            steps {
                script {
                    env.IMAGE_TAG = "rimandeepsingh/amazon:${BUILD_NUMBER}"

                    // Optional cleanup
                    sh "docker rmi -f amazon ${env.IMAGE_TAG} || true"

                    sh "docker build -t amazon ."
                }
            }
        }

        stage("Tag & Push to DockerHub") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker-cred', variable: 'dockerpwd')]) {
                        sh "docker login -u rimandeepsingh -p ${dockerpwd}"
                        sh "docker tag amazon ${env.IMAGE_TAG}"
                        sh "docker push ${env.IMAGE_TAG}"

                        // Also push latest
                        sh "docker tag amazon rimandeepsingh/amazon:latest"
                        sh "docker push rimandeepsingh/amazon:latest"
                    }
                }
            }
        }

        stage("Trivy Scan Image") {
            steps {
                script {
                    sh """
                    echo '🔍 Running Trivy scan on ${env.IMAGE_TAG}'

                    # JSON report
                    trivy image -f json -o trivy-image.json ${env.IMAGE_TAG}

                    # HTML report using built-in HTML format
                    trivy image -f table -o trivy-image.txt ${env.IMAGE_TAG}

                    # Fail build if HIGH/CRITICAL vulnerabilities found
                    # trivy image --exit-code 1 --severity HIGH,CRITICAL ${env.IMAGE_TAG} || true
                """
                }
            }
        }

        stage("Deploy to Container") {
            steps {
                script {
                    sh "docker rm -f amazon || true"
                    sh "docker run -d --name amazon -p 80:80 ${env.IMAGE_TAG}"
                }
            }
        }
    }

    post {
        success {
            script {
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: ' Github User'
                emailext(
                    subject: "Pipeline SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is a Jenkins Amazon CICD pipeline status.</p>
                        <p>Project: ${env.JOB_NAME}</p>
                        <p>Build Number: ${env.BUILD_NUMBER}</p>
                        <p>Build Status: SUCCESS</p>
                        <p>Started by: ${buildUser}</p>
                        <p>Build URL: <a href=\"${env.BUILD_URL}\">${env.BUILD_URL}</a></p>
                    """,
                    to: 'rimandeep267@gmail.com',
                    mimeType: 'text/html',
                    attachLog: true,
                    compressLog: true,
                    attachmentsPattern: 'trivyfs.txt,trivy-image.json,trivy-image.txt,dependency-check-report.xml'
                )
            }
        }
        failure {
            script {
                def buildUser = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')[0]?.userId ?: ' Github User'
                emailext(
                    subject: "Pipeline FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    body: """
                        <p>This is a Jenkins Amazon CICD pipeline status.</p>
                        <p>Project: ${env.JOB_NAME}</p>
                        <p>Build Number: ${env.BUILD_NUMBER}</p>
                        <p>Build Status: FAILURE</p>
                        <p>Started by: ${buildUser}</p>
                        <p>Build URL: <a href=\"${env.BUILD_URL}\">${env.BUILD_URL}</a></p>
                    """,
                    to: 'rimandeep267@gmail.com',
                    mimeType: 'text/html',
                    attachLog: true,
                    compressLog: true,
                    attachmentsPattern: 'trivyfs.txt,trivy-image.json,trivy-image.txt,dependency-check-report.xml'
                )
            }
        }
    }
}




