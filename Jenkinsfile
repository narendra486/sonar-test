pipeline {
    agent any

    environment {
        SONAR_HOST_URL = "http://167.86.125.122:1338"
        GITHUB_API_URL = "https://api.github.com"
        // Your GitHub personal access token credential ID configured in Jenkins
        GITHUB_CREDENTIALS_ID = 'github-token'
    }

    stages {
        stage('SonarQube Analysis - Fixed') {
            environment {
                scannerHome = tool 'sonar-scanner'  // Use your configured SonarScanner tool name
            }
            steps {
                script {
                    echo "🔍 Checking SonarQube container..."
                    def running = sh(script: "docker ps --format '{{.Names}}' | grep -w sonarqube || true", returnStdout: true).trim()
                    if (!running) {
                        sh 'docker compose up -d sonarqube'
                        sleep 30
                    }
                    sh "curl -I ${SONAR_HOST_URL} || true"
                }

                withSonarQubeEnv('SonarQube Server') {
                    sh """
                        ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=bwapp-scan \
                            -Dsonar.projectName='bWAPP Scan' \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.exclusions=**/node_modules/**,**/vendor/**,**/*.zip,**/*.jar \
                            -Dsonar.sourceEncoding=UTF-8 \
                            -Dsonar.php.coverage.reportPaths=coverage.xml \
                            -Dsonar.python.version=3.10
                    """
                }

                script {
                    if (fileExists('.scannerwork/report-task.txt')) {
                        timeout(time: 60, unit: 'MINUTES') {
                            def qualityGate = waitForQualityGate(abortPipeline: false)
                            if (qualityGate.status != 'OK') {
                                currentBuild.result = 'FAILURE'
                            }

                            // Notify GitHub about quality gate status
                            def commitSha = env.GIT_COMMIT ?: env.CHANGE_TARGET
                            def githubApi = "${env.GITHUB_API_URL}/repos/${env.GITHUB_REPOSITORY}/statuses/${commitSha}"
                            def githubToken = credentials(env.GITHUB_CREDENTIALS_ID)

                            def state = (qualityGate.status == 'OK') ? 'success' : 'failure'
                            def description = "SonarQube Quality Gate: ${qualityGate.status}"
                            def context = "SonarQube Quality Gate"

                            if (commitSha && env.GITHUB_REPOSITORY) {
                                def payload = new groovy.json.JsonBuilder([
                                    state: state,
                                    description: description,
                                    context: context
                                ]).toString()

                                sh """
                                curl -s -X POST -H "Authorization: token ${githubToken}" -H "Content-Type: application/json" \\
                                -d '${payload}' \\
                                ${githubApi}
                                """
                                echo "✅ GitHub status updated: ${state}"
                            } else {
                                echo "⚠️ Not updating GitHub status: missing commit SHA or repository info"
                            }

                            if (qualityGate.status != 'OK') {
                                error("Quality Gate failed with status: ${qualityGate.status}")
                            }
                        }
                    } else {
                        error("⚠️ report-task.txt not found - skipping quality gate wait and failing the build")
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully with passing Quality Gate."
        }
        failure {
            echo "❌ Pipeline failed due to Quality Gate failure or SonarQube scan issues."
        }
    }
}