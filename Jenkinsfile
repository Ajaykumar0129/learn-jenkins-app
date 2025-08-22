pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = 'a9dcaa70-ea68-4efa-9061-eece350ef231'
        NETLIFY_AUTH_TOKEN = credentials('Netlify token')
        REACT_APP_VERSION = "1.0.$BUILD_ID"
    }

    stages {
        stage('AWS-CLI') {
            agent {
                docker {
                    image 'amazon/aws-cli'
                    args "--entrypoint=''"
                }
            }
            steps {
                sh '''
                     aws --version 
                     aws s3 ls
                   '''
            }
        }
        // stage('Docker') {
        //    steps {
        //       script {
        //          def img = docker.build("my-playwright")
        //         }
        //     }
        // }

        stage('Build') {
            agent {
                docker {
                    image 'node:22-alpine'
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
                            image 'node:22-alpine'
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
                            image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test  --reporter=html
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy Stage') {
            agent {
                docker {
                    image 'node:22-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --no-build --json > deploy-output.json
                    CI_ENVIRONMENT_URL=$(node-jq -r '.deploy_url' deploy-output.json)
                    npx playwright test  --reporter=html
                    
                '''
                script{
                    env.STAGING_URL=sh(script:"node_modules/.bin/node-jq -r '.deploy_url' deploy-output.json",returnStdout:true)
                 }
            }
           
        }
 
        stage('Stage E2E') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
               CI_ENVIRONMENT_URL = "${STAGING_URL}"
            }

                steps {
                        sh '''
                            serve -s build &
                            sleep 10
                            npx playwright test  --reporter=html
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
        }

        stage('Approval'){
            steps{
                timeout(activity: true, time: 1) {
                           input message: 'Proceed to prod deployment', ok: 'Looks good. Proceed the build'
                    }
            }
        }

         stage('Deploy Prod') {
            agent {
                docker {
                    image 'node:22-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --no-build --prod
                '''
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.54.0-noble'
                    reuseNode true
                }
            }

            environment {
               CI_ENVIRONMENT_URL = "https://689b17f8051008e951434aae--cozy-halva-50234e.netlify.app"
            }

                steps {
                        sh '''
                            npx playwright test  --reporter=html
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
        }
    }
     post {
        always {
            emailext(
                to: 'ajaykumar012997@gmail.com',
                subject: "Build ${currentBuild.currentResult}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
                Hello Team,
                
                The build result is: ${currentBuild.currentResult}
                Job: ${env.JOB_NAME}
                Build Number: ${env.BUILD_NUMBER}
                URL: ${env.BUILD_URL}
                """
            )
        }
    }
}
