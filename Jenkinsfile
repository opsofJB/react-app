pipeline {
    agent any

    environment {
        JAVA_HOME = "${WORKSPACE}/java21"
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }

    stages {
        stage('Download Java 21') {
            steps {
                sh '''
                    echo "Downloading Amazon Corretto 21..."
                    curl -L -o corretto.tar.gz https://corretto.aws/downloads/latest/amazon-corretto-21-x64-linux-jdk.tar.gz
                    mkdir -p java21
                    tar -xzf corretto.tar.gz -C java21 --strip-components=1
                    rm corretto.tar.gz
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "Java version:"
                    java -version
                    ./gradlew build --no-daemon
                '''
            }
      } 

        stage('Build') {
            steps {
                echo 'Running build automation'
                sh 'chmod +x gradlew'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'build/dist/reactApp'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    app = docker.build("imageofJB/react-app")
                    app.inside {
                        sh 'echo $(curl localhost:1233)'
                    }
                }
            }
        }
        stage('Push Docker Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    docker.withRegistry('https://registry.hub.docker.com', 'dockerhub_login') {
                        app.push("${env.BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }
         stage('DeployToStaging') {
            when {
                branch 'main'
            }
            steps {
                    script {
                        sh "docker pull imageofJB/react-app:${env.BUILD_NUMBER}"
                        try {
                            sh "docker stop react-app"
                            sh "docker rm react-app"
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "docker run --restart always --name react-app -p 1233:80 -d imageofJB/react-app:${env.BUILD_NUMBER}"
                    }
            }
        }
        
        stage("Check HTTP Response") {
            steps {
                script {
                    final String url = "http://localhost:1233"
                    
                    final String response = sh(script: "curl -o /dev/null -s -w '%{http_code}\\n' $url", returnStdout: true).trim()
                    
                    if (response == "200") {
                        echo response
                        println "Successful Response Code" 
                    } else {
                        echo response
                        println "Error Response Code" 
                    }

                }
            }
        }
        
        stage('DeployToProduction') {
            when {
                branch 'main'
            }
            steps {
                input 'Does the staging environment look OK? Did You get 200 response?'
                 milestone(1)
                    script {
                        sh "docker pull imageofJB/react-app:${env.BUILD_NUMBER}"
                        try {
                            sh "docker stop react-app"
                            sh "docker rm react-app"
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "docker run --restart always --name react-app -p 1233:80 -d imageofJB/react-app:${env.BUILD_NUMBER}"
                    }
            }
        }
    }
}
