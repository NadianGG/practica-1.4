pipeline {
    agent any

    environment {
        REPO_URL        = 'https://github.com/NadianGG/practica-1.4.git'
        CONFIG_REPO_URL = 'https://github.com/NadianGG/todo-list-aws-config.git'

        // Pipeline CI (Reto 4)
        BRANCH         = 'develop'   // repo código
        CONFIG_BRANCH  = 'staging'   // repo config (rama)
        CONFIG_ENV     = 'staging'   // sam deploy --config-env
    }

    stages {

        stage('Get Code') {
            steps {
                echo "Clonando repositorio de código (${BRANCH})..."
                checkout([$class: 'GitSCM',
                    branches: [[name: "*/${BRANCH}"]],
                    userRemoteConfigs: [[url: "${REPO_URL}"]]
                ])

                echo "Descargando configuración (${CONFIG_BRANCH})..."
                sh '''
                    rm -rf config-repo
                    git clone -b "${CONFIG_BRANCH}" "${CONFIG_REPO_URL}" config-repo
                    cp config-repo/samconfig.toml .
                    
                    # Quitar s3_bucket para poder usar --resolve-s3 (evita colisión y bucket inexistente)
                    sed -i '/^s3_bucket\\s*=\\s*/d' samconfig.toml
                    
                    echo "samconfig.toml cargado:"
                    cat samconfig.toml
                '''
            }
        }

        stage('Static Test') {
            steps {
                echo "Ejecutando análisis estático en nodo: $NODE_NAME"
                sh '''
                    cd src
                    python3 -m flake8 . --output-file=../flake8-report.txt || true
                    bandit -r . -f json -o ../bandit-report.json || true
                '''
                archiveArtifacts artifacts: 'flake8-report.txt, bandit-report.json', allowEmptyArchive: true
            }
        }

        stage('Deploy') {
            steps {
                script {
                    env.SAM_REGION = sh(
                        script: """
                            python3 -c "import tomllib,os; e=os.environ.get('CONFIG_ENV','staging'); \
                            cfg=tomllib.load(open('samconfig.toml','rb')); \
                            print(cfg[e]['deploy']['parameters']['region'])"
                        """,
                        returnStdout: true
                    ).trim()
                
                    env.STACK_NAME = sh(
                        script: """
                            python3 -c "import tomllib,os; e=os.environ.get('CONFIG_ENV','staging'); \
                            cfg=tomllib.load(open('samconfig.toml','rb')); \
                            print(cfg[e]['deploy']['parameters']['stack_name'])"
                        """,
                        returnStdout: true
                    ).trim()
                }

                echo "Región (samconfig): ${env.SAM_REGION}"
                echo "Stack (samconfig): ${env.STACK_NAME}"

                sh '''
                    sam build
                    sam validate --region "${SAM_REGION}"
                    sam deploy \
                      --config-env "${CONFIG_ENV}" \
                      --resolve-s3 \
                      --no-confirm-changeset \
                      --no-fail-on-empty-changeset
                '''
            }
        }

        stage('Rest Test') {
            steps {
                echo "Ejecutando tests REST en nodo: $NODE_NAME"

                script {
                    env.BASE_URL = sh(
                        script: '''
                        aws cloudformation describe-stacks \
                            --region "${SAM_REGION}" \
                            --stack-name "${STACK_NAME}" \
                            --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                            --output text
                        ''',
                        returnStdout: true
                    ).trim()
                }

                echo "Endpoint detectado: ${env.BASE_URL}"

                sh '''
                    export BASE_URL=${BASE_URL}
                    pytest test/integration/todoApiTest.py \
                        --maxfail=1 \
                        --disable-warnings
                '''
            }
        }

        stage('Promote') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                sh '''
                    git config user.name "Jenkins"
                    git config user.email "jenkins@ci.local"

                    # Limpieza de archivos externos (config)
                    rm -f samconfig.toml || true
                    rm -rf config-repo || true

                    git fetch origin
                    git checkout master
                    git merge origin/develop --no-ff -m "Release desde Jenkins"

                    # mantener Jenkinsfile del master
                    git checkout origin/master -- Jenkinsfile

                    git push https://$GITHUB_TOKEN@github.com/NadianGG/practica-1.4.git master
                '''
                }
            }
        }
    }

    post {
        success { echo 'Pipeline CI completado con éxito.' }
        failure { echo 'Pipeline CI falló.' }
    }
}
