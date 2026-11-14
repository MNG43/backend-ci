// CI backend SamaFile : tests Pytest, scans Python, image Docker, Trivy, Docker Hub.
// Job Jenkins : Pipeline script from SCM, script path Jenkinsfile.
// Credential : dockerhub-credentials (utilisateur + jeton Docker Hub).

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(name: 'PUBLISH', defaultValue: false, description: 'Publier l image sur Docker Hub après les scans')
    }

    environment {
        LOCAL_IMAGE = 'samafile-backend:ci'
        CI_IMAGE = 'samafile-backend-ci:build'
        TRIVY_IMAGE = 'aquasec/trivy:0.58.1'
    }

    stages {
        stage('Préparation') {
            steps {
                sh 'mkdir -p reports'
                sh 'docker build -f Dockerfile.ci -t "$CI_IMAGE" .'
            }
        }

        stage('Tests Pytest') {
            steps {
                sh '''
                set -eu
                docker network create "samafile-ci-${BUILD_NUMBER}"
                docker rm -f "samafile-pg-${BUILD_NUMBER}" >/dev/null 2>&1 || true
                docker run -d --name "samafile-pg-${BUILD_NUMBER}" \
                    --network "samafile-ci-${BUILD_NUMBER}" \
                    -e POSTGRES_USER=samafile \
                    -e POSTGRES_PASSWORD=ci_only \
                    -e POSTGRES_DB=samafile_test \
                    postgres:16-alpine
                ready=0
                for _ in $(seq 1 30); do
                    if docker exec "samafile-pg-${BUILD_NUMBER}" pg_isready -U samafile -d samafile_test; then
                        ready=1
                        break
                    fi
                    sleep 2
                done
                test "$ready" = 1
                docker create --name "samafile-tests-${BUILD_NUMBER}" \
                    --network "samafile-ci-${BUILD_NUMBER}" \
                    -e APP_ENV=test \
                    -e TEST_DATABASE_ALLOW_REMOTE=1 \
                    -e SECRET_KEY=ci-secret-key-not-for-production-use \
                    -e "TEST_DATABASE_URL=postgresql+psycopg://samafile:ci_only@samafile-pg-${BUILD_NUMBER}:5432/samafile_test" \
                    "$CI_IMAGE" \
                    python -m pytest tests --junitxml=reports/pytest.xml
                set +e
                docker start -a "samafile-tests-${BUILD_NUMBER}"
                test_code=$?
                set -e
                docker cp "samafile-tests-${BUILD_NUMBER}:/app/reports/pytest.xml" reports/pytest.xml || true
                docker rm -f "samafile-tests-${BUILD_NUMBER}" >/dev/null
                exit "$test_code"
                '''
            }
        }

        stage('SAST Bandit') {
            steps {
                sh '''
                set -eu
                docker create --name "samafile-bandit-${BUILD_NUMBER}" "$CI_IMAGE" \
                    bandit -r src -ll -f json -o reports/bandit.json
                set +e
                docker start -a "samafile-bandit-${BUILD_NUMBER}"
                scan_code=$?
                set -e
                docker cp "samafile-bandit-${BUILD_NUMBER}:/app/reports/bandit.json" reports/bandit.json || true
                docker rm -f "samafile-bandit-${BUILD_NUMBER}" >/dev/null
                if [ "$scan_code" -ne 0 ]; then exit "$scan_code"; fi
                docker run --rm "$CI_IMAGE" bandit -r src -ll
                '''
            }
        }

        stage('Dépendances Python') {
            steps {
                sh '''
                set -eu
                docker create --name "samafile-audit-${BUILD_NUMBER}" "$CI_IMAGE" \
                    pip-audit -r requirements.txt --progress-spinner off --format json --output reports/pip-audit.json
                set +e
                docker start -a "samafile-audit-${BUILD_NUMBER}"
                audit_code=$?
                set -e
                docker cp "samafile-audit-${BUILD_NUMBER}:/app/reports/pip-audit.json" reports/pip-audit.json || true
                docker rm -f "samafile-audit-${BUILD_NUMBER}" >/dev/null
                if [ "$audit_code" -ne 0 ]; then exit "$audit_code"; fi
                docker run --rm "$CI_IMAGE" pip-audit -r requirements.txt --progress-spinner off
                '''
            }
        }

        stage('Hadolint') {
            steps {
                sh '''
                set -eu
                docker run --rm -i hadolint/hadolint:v2.12.0-alpine hadolint --failure-threshold error - < Dockerfile
                '''
            }
        }

        stage('Image Docker') {
            steps {
                sh 'docker build -t "$LOCAL_IMAGE" .'
            }
        }

        stage('Trivy image et sources') {
            steps {
                sh '''
                set -eu
                run_trivy() {
                    docker run --rm "$@"
                }
                if run_trivy \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    "$TRIVY_IMAGE" image \
                    --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
                    --format table \
                    "$LOCAL_IMAGE" > reports/trivy-image.txt 2>&1; then
                    image_code=0
                else
                    image_code=$?
                fi
                echo "----- Trivy image -----"
                cat reports/trivy-image.txt || true
                if run_trivy \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    "$TRIVY_IMAGE" image \
                    --scanners vuln,secret,misconfig \
                    --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 \
                    --format table \
                    "$CI_IMAGE" > reports/trivy-fs.txt 2>&1; then
                    fs_code=0
                else
                    fs_code=$?
                fi
                echo "----- Trivy fs -----"
                cat reports/trivy-fs.txt || true
                test "$image_code" = 0
                test "$fs_code" = 0
                '''
            }
        }

        stage('Publication Docker Hub') {
            when {
                expression { return params.PUBLISH }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                    set -eu
                    echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin
                    image="${DOCKERHUB_USER}/samafile-backend"
                    docker tag "$LOCAL_IMAGE" "${image}:${BUILD_NUMBER}"
                    docker push "${image}:${BUILD_NUMBER}"
                    sha="$(git rev-parse --short HEAD 2>/dev/null || true)"
                    if [ -n "$sha" ]; then
                        docker tag "$LOCAL_IMAGE" "${image}:${sha}"
                        docker push "${image}:${sha}"
                    fi
                    branch="${BRANCH_NAME:-${GIT_BRANCH:-}}"
                    if printf '%s' "$branch" | grep -Eq '(^|/)(main|master|develop)$'; then
                        docker tag "$LOCAL_IMAGE" "${image}:latest"
                        docker push "${image}:latest"
                    fi
                    docker logout
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f "samafile-pg-${BUILD_NUMBER}" >/dev/null 2>&1 || true
                docker rm -f "samafile-tests-${BUILD_NUMBER}" "samafile-bandit-${BUILD_NUMBER}" "samafile-audit-${BUILD_NUMBER}" >/dev/null 2>&1 || true
                docker network rm "samafile-ci-${BUILD_NUMBER}" >/dev/null 2>&1 || true
            '''
            junit allowEmptyResults: true, testResults: 'reports/pytest.xml'
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
        }
    }
}
