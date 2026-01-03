pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'f83d46aa-7eb4-454a-904d-39190421ca14'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
        // AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        // AWS_SECRET_ACCESS_KEY = credentials('FiGbznyYwmP8cJl8KevClzML+a1EHZFzz0B6O22H')
    }

    stages {

        stage('AWS') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    args "--entrypoint=''"
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
                    // some block
                    sh '''
                        aws --version
                        echo "Hello, Jenkins!" > index.html
                        aws s3 cp index.html s3://learn-jenkins-2026/test.txt
                    '''
                }
            }
        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            #test -f build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'my-playwright'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            serve -s build &
                            sleep 10
                            echo "Waiting for server to be ready..."
                            timeout 60 bash -c 'until curl -f http://localhost:3000 > /dev/null 2>&1; do sleep 1; done' || echo "Warning: Server check timed out"
                            npx playwright test  --reporter=html
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --json > deploy-output.json
                    jq -r '.deploy_url' deploy-output.json
                '''
                script {
                    env.STAGING_URL = sh(script: "jq -r '.deploy_url' deploy-output.json", returnStdout: true).trim()
                }
            }
        }

        stage('Staging E2E') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }

            steps {
                sh '''
                    npx playwright test  --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        // stage('Approval') {
        //     steps {
        //         timeout(time: 15, unit: 'MINUTES') {
        //             input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
        //         }
        //     }
        // }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod
                '''
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://magnificent-kelpie-0179f8.netlify.app'
            }

            steps {
                sh '''
                    npx playwright test  --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
