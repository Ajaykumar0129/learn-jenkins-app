pipeline {
    agent any

    stages {
        stage('Build') {
            agent{
                docker{
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
         stage('Test') {
            agent{
                docker{
                    image 'node:22-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                   test -f build/index.html && echo "✅ File exists" || (echo "❌ File not found" && exit 1)
                   npm test
                '''
            }
        }
    }
    post{
        always{
            junit 'test-results/junit.xml'
        }
    }
}
