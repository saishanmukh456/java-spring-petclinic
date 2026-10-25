pipeline {
    agent any

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://https://github.com/saishanmukh456/java-spring-petclinic.git'
            }
        }

        stage('Build and Test') {
            steps {
                sh 'mvn -B clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        mvn -B org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar \
                          -Dsonar.projectKey=spring-petclinic \
                          -Dsonar.projectName="Spring PetClinic"
                    '''
                }
            }
        }

        stage('Test SSH Authentication') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'petclinic-deploy-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        set -eu

                        echo "Testing SSH authentication to application server..."

                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            "$SSH_USER@44.201.134.73" \
                            'echo SSH_OK'
                    '''
                }
            }
        }

        stage('Automatic Deployment') {
            steps {
                withCredentials([
                    sshUserPrivateKey(
                        credentialsId: 'petclinic-deploy-key',
                        keyFileVariable: 'SSH_KEY',
                        usernameVariable: 'SSH_USER'
                    )
                ]) {
                    sh '''
                        set -eu

                        JAR_FILE="$(find target -maxdepth 1 -type f \
                            -name "*.jar" \
                            ! -name "*-sources.jar" \
                            ! -name "*-javadoc.jar" \
                            -print -quit)"

                        if [ -z "$JAR_FILE" ]; then
                            echo "ERROR: No application JAR found in target/"
                            exit 1
                        fi

                        echo "Deploying artifact: $JAR_FILE"

                        scp -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            "$JAR_FILE" \
                            "$SSH_USER@44.201.134.73:/home/deploy/app.jar"

                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            "$SSH_USER@44.201.134.73" \
                            'sudo /usr/local/sbin/deploy-petclinic'

                        echo "Deployment command completed."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Build, tests, SonarQube analysis, SSH authentication, and deployment completed.'
        }
        failure {
            echo 'FAILED: Check Console Output to identify the failed stage.'
        }
    }
}
