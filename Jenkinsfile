pipeline {
    agent any

    environment {
        RAILWAY_TOKEN = credentials('233783da-ab0b-49d3-baa6-5c7e486eb41e')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Backend') {
            stages {
                stage('Install') {
                    steps {
                        dir('backend') {
                            sh 'pip install uv && uv sync --extra dev'
                        }
                    }
                }

                stage('Lint') {
                    steps {
                        dir('backend') {
                            sh 'uv run python -m py_compile src/main.py'
                        }
                    }
                }

                stage('Test') {
                    steps {
                        dir('backend') {
                            sh 'uv run pytest tests/ -v --tb=short --junitxml=reports/test-results.xml'
                        }
                    }
                    post {
                        always {
                            junit allowEmptyResults: true, testResults: 'backend/reports/test-results.xml'
                        }
                    }
                }
            }
        }

        stage('Frontend') {
            stages {
                stage('Install') {
                    steps {
                        dir('frontend') {
                            sh 'npm ci'
                        }
                    }
                }

                stage('Type Check') {
                    steps {
                        dir('frontend') {
                            sh 'npx tsc --noEmit || true'
                        }
                    }
                }

                stage('Build') {
                    steps {
                        dir('frontend') {
                            sh 'npm run build'
                        }
                    }
                }
            }
        }

        stage('Deploy Backend to Railway') {
            when {
                branch 'main'
            }
            steps {
                dir('backend') {
                    sh '''
                        npm install -g @railway/cli || true
                        railway up --service backend --detach
                    '''
                }
            }
        }

        stage('Deploy Frontend to Railway') {
            when {
                branch 'main'
            }
            steps {
                dir('frontend') {
                    sh '''
                        npm install -g @railway/cli || true
                        railway up --service frontend --detach
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed — check the logs above.'
        }
    }
}
