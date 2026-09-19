pipeline {
    agent any

    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:$PATH"
        IMAGE_NAME = 'goof'
        TEST_CONTAINER = 'goof-testing'
        PROD_CONTAINER = 'goof-production'
        TEST_PORT = '3001'
        PROD_PORT = '3002'
        IMAGE_TAG = "goof:build-${BUILD_NUMBER}"
        RELEASE_TAG = "goof:release-${BUILD_NUMBER}"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
    }

    stages {

        // Build Stage
        stage('Build') {
            steps {
                sh '''
                    set -e
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    docker build --tag "$IMAGE_TAG" .

                    cat > build-info.txt <<EOF
                    Application: Goof
                    Build Number: $BUILD_NUMBER
                    Git Commit: ${GIT_COMMIT:-unknown}
                    Docker Image: $IMAGE_TAG
                    Build Time: $(date)
                    EOF
                '''

                archiveArtifacts artifacts: 'build-info.txt', fingerprint: true
            }
        }

        // Test Stage

        stage('Test') {
            steps {
                sh '''
                    set -e
                    node -e '
                    const assert = require("assert");
                    const fs = require("fs");

                    const packageJson = JSON.parse(
                      fs.readFileSync("package.json", "utf8")
                    );

                    assert(packageJson.name);
                    assert(packageJson.version);
                    assert(packageJson.scripts);
                    assert(packageJson.scripts.start);
                    assert(fs.existsSync("app.js"));
                    assert(fs.existsSync("Dockerfile"));

                    console.log("Application tests passed.");
                    '

                    docker network create goof-test-network 2>/dev/null || true

                    docker rm -f goof-mongo goof-mysql "$TEST_CONTAINER" 2>/dev/null || true

                    docker run \
                          --detach \
                          --name goof-mongo \
                          --network goof-test-network \
                          mongo:4.4

                    docker run \
                          --platform linux/amd64 \
                          --detach \
                          --name goof-mysql \
                          --network goof-test-network \
                          --env MYSQL_ROOT_PASSWORD=root \
                          --env MYSQL_DATABASE=acme \
                          mysql:5.7

                    docker run \
                          --detach \
                          --name "$TEST_CONTAINER" \
                          --network goof-test-network \
                          --env DOCKER=1 \
                          --publish "$TEST_PORT:3001" \
                         "$IMAGE_TAG"

                    for i in $(seq 1 30); do
                          if curl --silent --fail \
                              --max-time 3 \
                              "http://localhost:$TEST_PORT/" > /dev/null; then
                              echo "Integration test passed."
                              break
                          fi

                          if [ "$i" -eq 30 ]; then
                              echo "Integration test failed."
                              docker logs "$TEST_CONTAINER" || true
                              docker logs goof-mysql || true
                              docker logs goof-mongo || true
                              exit 1
                          fi

                          sleep 2
                    done 

                    docker rm -f "$TEST_CONTAINER" goof-mysql goof-mongo
                    docker network rm goof-test-network
              '''
            }
        }
        

        // Code Quality Stage
        stage('Code Quality') {
    steps {
        script {
            def scannerHome = tool 'SonarScanner'

            withSonarQubeEnv('SonarQube') {
                sh """
                    ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=nodejs-goof-devops-pipeline-HD \
                        -Dsonar.projectName=nodejs-goof-devops-pipeline-HD \
                        -Dsonar.sources=. \
                        -Dsonar.exclusions=node_modules/**,public/js/bundle.js,tests/**,exploits/**
                """
            }
        }

        timeout(time: 5, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}

        // Security Stage
        stage('Security') {
            steps {
                sh '''
                    set +e

                    ./node_modules/.bin/snyk test \
                        --json-file-output=snyk-report.json

                    SNYK_EXIT=$?

                    echo "Snyk exit code: $SNYK_EXIT"

                    if [ -f snyk-report.json ]; then
                        echo "Snyk report generated."
                    fi

                    exit 0
                '''

                archiveArtifacts artifacts: 'snyk-report.json',
                    allowEmptyArchive: true,
                    fingerprint: true
            }
        }

        // Deploy Stage
        stage('Deploy') {
            steps {
                sh '''
                    set -e

                    docker rm -f "$TEST_CONTAINER" 2>/dev/null || true

                    docker run \
                        --detach \
                        --restart unless-stopped \
                        --name "$TEST_CONTAINER" \
                        --publish "$TEST_PORT:3001" \
                        "$IMAGE_TAG"

                    for i in $(seq 1 30); do
                        if curl --silent --fail \
                            --max-time 3 \
                            "http://localhost:$TEST_PORT/" > /dev/null; then
                            echo "Testing environment deployed successfully."
                            exit 0
                        fi

                        if [ "$i" -eq 30 ]; then
                            echo "Deployment failed."
                            docker logs "$TEST_CONTAINER" || true
                            docker rm -f "$TEST_CONTAINER" || true
                            exit 1
                        fi

                        sleep 2
                    done
                '''
            }
        }

        // Release Stage
        stage('Release') {
            steps {
                sh '''
                    set -e

                    docker tag "$IMAGE_TAG" "$RELEASE_TAG"

                    docker rm -f "$PROD_CONTAINER" 2>/dev/null || true

                    docker run \
                        --detach \
                        --restart unless-stopped \
                        --name "$PROD_CONTAINER" \
                        --publish "$PROD_PORT:3001" \
                        "$RELEASE_TAG"

                    for i in $(seq 1 30); do
                        if curl --silent --fail \
                            --max-time 3 \
                            "http://localhost:$PROD_PORT/" > /dev/null; then
                            echo "Production release successful."
                            break
                        fi

                        if [ "$i" -eq 30 ]; then
                            echo "Production release failed."
                            docker logs "$PROD_CONTAINER" || true
                            docker rm -f "$PROD_CONTAINER" || true
                            exit 1
                        fi

                        sleep 2
                    done

                    cat > release-info.txt <<EOF
                    Application: Goof
                    Release: $RELEASE_TAG
                    Source Build: $IMAGE_TAG
                    Build Number: $BUILD_NUMBER
                    Environment: Production
                    Release Time: $(date)
                    EOF
                '''

                archiveArtifacts artifacts: 'release-info.txt', fingerprint: true
            }
        }

        // Monitoring Stage
        stage('Monitoring') {
            steps {
                sh '''
                    set -e

                    if ! docker ps \
                        --filter "name=$PROD_CONTAINER" \
                        --filter "status=running" \
                        --format '{{.Names}}' |
                        grep -q "^${PROD_CONTAINER}$"; then
                        echo "ALERT: Production container is not running."
                        exit 1
                    fi

                    HTTP_STATUS=$(curl \
                        --silent \
                        --output /dev/null \
                        --write-out "%{http_code}" \
                        --max-time 5 \
                        "http://localhost:$PROD_PORT/")

                    echo "Production HTTP status: $HTTP_STATUS"

                    if [ "$HTTP_STATUS" -ge 200 ] && [ "$HTTP_STATUS" -lt 400 ]; then
                        echo "Monitoring check passed."
                    else
                        echo "ALERT: Production health check failed."
                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        success {
            echo 'All seven pipeline stages completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage and console output.'
        }

        always {
            sh '''
                docker rm -f "$TEST_CONTAINER" 2>/dev/null || true
            '''
        }
    }
}
