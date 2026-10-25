
pipeline {
    agent any

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
                            "$SSH_USER@54.164.62.109:/home/deploy/app.jar"

                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            "$SSH_USER@54.164.62.109" \
                            'sudo /usr/local/sbin/deploy-petclinic'

                        echo "Deployment command completed."
                    '''
                }
            }
        }

        stage('Post-Deployment Verification') {
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

                        echo "Checking application service status..."

                        ssh -i "$SSH_KEY" \
                            -o BatchMode=yes \
                            -o StrictHostKeyChecking=yes \
                            -o ConnectTimeout=10 \
                            "$SSH_USER@54.164.62.109" \
                            'sudo systemctl is-active --quiet petclinic.service'

                        echo "Service is active."

                        echo "Checking application health endpoint..."

                        APP_HEALTH_URL="http:"'//54.164.62.109:8080/actuator/health'

                        ATTEMPTS=12
                        COUNT=1

                        while [ "$COUNT" -le "$ATTEMPTS" ]; do
                            if RESPONSE="$(curl --silent --show-error \
                                --fail --max-time 10 \
                                "$APP_HEALTH_URL" 2>/dev/null)" \
                                && printf '%s' "$RESPONSE" \
                                    | grep -q '"status"[[:space:]]*:[[:space:]]*"UP"'; then

                                echo "Health response: $RESPONSE"
                                echo "SUCCESS: Petclinic is healthy after deployment."
                                exit 0
                            fi

                            echo "Health check attempt $COUNT/$ATTEMPTS failed; retrying in 5 seconds..."
                            sleep 5
                            COUNT=$((COUNT + 1))
                        done

                        echo "ERROR: Petclinic health check failed after deployment."
                        exit 1
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Build, tests, SonarQube analysis, SSH authentication, automatic deployment, and post-deployment verification completed.'
        }

        failure {
            echo 'FAILED: One or more pipeline stages failed. Check Console Output to identify the issue.'
        }
    }
}
