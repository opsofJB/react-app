pipeline {
    agent any

    tools {
        jdk 'jdk17'
    }

    environment {
        JAVA_HOME = '/usr/lib/jvm/java-17-amazon-corretto'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
    }

    stages {

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
                sh 'sudo chmod +x gradlew'
                sh './gradlew build --no-daemon'
                archiveArtifacts artifacts: 'dist/reactApp'
            }
        }
        stage('Build Docker Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    app = docker.build("gcastill0/react-app")
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
                        sh "docker pull gcastill0/react-app:${env.BUILD_NUMBER}"
                        try {
                            sh "docker stop react-app"
                            sh "docker rm react-app"
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "docker run --restart always --name react-app -p 1233:80 -d gcastill0/react-app:${env.BUILD_NUMBER}"
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
                        sh "docker pull gcastill0/react-app:${env.BUILD_NUMBER}"
                        try {
                            sh "docker stop react-app"
                            sh "docker rm react-app"
                        } catch (err) {
                            echo: 'caught error: $err'
                        }
                        sh "docker run --restart always --name react-app -p 1233:80 -d gcastill0/react-app:${env.BUILD_NUMBER}"
                    }
            }
        }
    }
}
