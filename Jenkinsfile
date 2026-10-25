
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/saishanmukh456/java-spring-petclinic.git'
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
                        set -eu

                        mvn -B \
                          org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar \
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

                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            "$SSH_USER@54.164.62.109" \
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

                        JAR_FILE="target/spring-petclinic-4.0.0-SNAPSHOT.jar"

                        if [ ! -f "$JAR_FILE" ]; then
                            echo "ERROR: Expected application JAR not found: $JAR_FILE"
                            echo "Available JAR files:"
                            find target -maxdepth 1 -type f -name "*.jar" -print
                            exit 1
                        fi

                        echo "Deploying $JAR_FILE"

                        scp -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            "$JAR_FILE" \
                            "$SSH_USER@54.164.62.109:/home/deploy/app.jar"

                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            "$SSH_USER@54.164.62.109" \
                            'sudo /usr/local/sbin/deploy-petclinic'

                        echo "Automatic deployment command completed."
                    '''
                }
            }
        }

        stage('Post-Deployment Verification') {
            steps {
                sh '''
                    set -eu

                    APP_HEALTH_URL="http:"'//54.164.62.109:8080/actuator/health'

                    ATTEMPTS=12
                    COUNT=1

                    while [ "$COUNT" -le "$ATTEMPTS" ]; do
                        echo "Health check attempt $COUNT/$ATTEMPTS"

                        if RESPONSE="$(curl --silent --show-error \
                            --fail --max-time 10 \
                            "$APP_HEALTH_URL" 2>/dev/null)" \
                            && printf '%s' "$RESPONSE" \
                                | grep -q '"status"[[:space:]]*:[[:space:]]*"UP"'; then

                            echo "Health response: $RESPONSE"
                            echo "SUCCESS: PetClinic is healthy."
                            exit 0
                        fi

                        if [ "$COUNT" -lt "$ATTEMPTS" ]; then
                            sleep 5
                        fi

                        COUNT=$((COUNT + 1))
                    done

                    echo "ERROR: Application health verification failed."
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Build, tests, SonarQube analysis, SSH authentication, automatic deployment, and health verification completed.'
        }

        failure {
            echo 'FAILED: Check the first failed stage in Jenkins Console Output.'
        }
    }
}
