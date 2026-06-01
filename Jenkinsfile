pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 1, unit: 'HOURS')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        GITHUB_REPO = 'https://github.com/mrunalchougule/jenkins.git'
        JAVA_HOME = '/usr/lib/jvm/java-11-openjdk-amd64'
        MAVEN_HOME = '/usr/share/maven'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${PATH}"
    }

    stages {
        stage('Checkout') {
            steps {
                echo '========== Checkout Stage =========='
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: "${GITHUB_REPO}"]]
                ])
                echo 'Repository cloned successfully!'
            }
        }

        stage('Build') {
            steps {
                echo '========== Build Stage =========='
                sh '''
                    java -version
                    mvn --version
                    mvn clean compile
                '''
                echo 'Build completed successfully!'
            }
        }

        stage('Test') {
            steps {
                echo '========== Test Stage =========='
                sh '''
                    mvn test
                '''
                echo 'Tests completed!'
            }
        }

        stage('Package') {
            steps {
                echo '========== Package Stage =========='
                sh '''
                    mvn package -DskipTests
                '''
                echo 'Packaging completed!'
            }
        }

        stage('Archive') {
            steps {
                echo '========== Archive Stage =========='
                archiveArtifacts artifacts: 'target/*.jar', allowEmptyArchive: true
                junit 'target/surefire-reports/*.xml'
            }
        }
    }

    post {
        always {
            echo '========== Cleanup =========='
            cleanWs()
        }
        success {
            echo '✓ Pipeline executed successfully!'
        }
        failure {
            echo '✗ Pipeline failed! Check logs for details.'
        }
    }
}
