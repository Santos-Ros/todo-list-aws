pipeline {
    agent none
    stages {
        stage('Get Code') {
            agent { label 'CP1.4-jenkins-02' }
            steps {
                withEnv(["PATH=/home/ubuntu/.local/bin:/usr/local/bin:${env.PATH}"]) {
                    cleanWs()
                    git credentialsId: 'github-token-id',
                        branch: 'develop',
                        url: 'https://github.com/Santos-Ros/todo-list-aws.git'
                    dir('config') {
                        deleteDir()
                    }
                    sh 'git clone --single-branch --branch staging https://github.com/Santos-Ros/todo-list-aws-config.git config'
                    sh 'cp config/samconfig.toml .'
                }
            }
        }
        stage('Static Test') {
            agent { label 'CP1.4-jenkins-02' }
            steps {
                withEnv(["PATH=/home/ubuntu/.local/bin:/usr/local/bin:${env.PATH}"]) {
                    echo 'Running flake8...'
                    sh 'flake8 --format=html --htmldir=flake8-report src/ || true'
                    echo 'Running bandit...'
                    sh 'bandit -r src/ -f html -o bandit-report.html || true'
                }
            }
            post {
                always {
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'flake8-report',
                        reportFiles: 'index.html',
                        reportName: 'Flake8 Report'
                    ])
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'bandit-report.html',
                        reportName: 'Bandit Report'
                    ])
                }
            }
        }
        stage('Deploy (Staging)') {
            agent { label 'CP1.4-jenkins-02' }
            steps {
                withEnv(["PATH=/home/ubuntu/.local/bin:/usr/local/bin:${env.PATH}"]) {
                    echo 'Building project with SAM...'
                    sh 'sam build'
                    echo 'Validating template...'
                    sh 'sam validate --region us-east-1'
                    echo 'Deploying to Staging environment...'
                    sh 'sam deploy --config-env staging --no-confirm-changeset || true'
                    sh '''
                        aws cloudformation describe-stacks \
                            --stack-name todo-list-aws-staging \
                            --region us-east-1 \
                            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                            --output text > api_url.txt
                    '''
                    echo 'Esperando que las Lambdas se inicialicen...'
                    sh 'sleep 30'
                    stash name: 'workspace-ci', includes: '**/*'
                }
            }
        }
        stage('REST Test') {
            agent { label 'CP1.4-jenkins-03' }
            steps {
                withEnv(["PATH=/home/ubuntu/.local/bin:/usr/local/bin:${env.PATH}"]) {
                    unstash 'workspace-ci'
                    script {
                        env.BASE_URL = readFile('api_url.txt').trim()
                    }
                    echo 'Running REST integration tests...'
                    sh 'pytest --junitxml=report.xml test/integration/todoApiTest.py'
                    echo 'Publicando resultados...'
                    junit 'report.xml'
                }
            }
        }
        stage('Promote') {
            agent { label 'CP1.4-jenkins-02' }
            steps {
                withEnv(["PATH=/home/ubuntu/.local/bin:/usr/local/bin:${env.PATH}"]) {
                    withCredentials([usernamePassword(credentialsId: 'github-token-id', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_TOKEN')]) {
                        echo 'Haciendo merge a master...'
                        sh '''
                            git config merge.ours.driver true
                            git config user.name "Santos Ros"
                            git config user.email "santosrospicazo@outlook.com"
                            git fetch --all
                            git checkout develop
                            git pull origin develop
                            git checkout master
                            git pull origin master
                            git merge develop --no-edit
                            git push https://Santos-Ros:${GIT_TOKEN}@github.com/Santos-Ros/todo-list-aws.git master
                        '''
                    }
                }
            }
        }
    }
}
