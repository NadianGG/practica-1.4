pipeline {
    agent any

    environment {
        REPO_URL        = 'https://github.com/NadianGG/practica-1.4.git'
        CONFIG_REPO_URL = 'https://github.com/NadianGG/todo-list-aws-config.git'

        // Pipeline CD (Reto 4)
        BRANCH        = 'master'       // repo código
        CONFIG_BRANCH = 'production'   // repo config (rama)
        CONFIG_ENV    = 'production'   // sam deploy --config-env
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

                    # Quitar s3_bucket para poder usar --resolve-s3 (evita colisión/nombre global)
                    sed -i '/^s3_bucket\\s*=\\s*/d' samconfig.toml

                    echo "samconfig.toml cargado:"
                    cat samconfig.toml
                '''
            }
        }

        stage('Deploy') {
            steps {
                script {
                    env.SAM_REGION = sh(
                        script: """
                            python3 -c "import tomllib,os; e=os.environ.get('CONFIG_ENV','production'); \
                            cfg=tomllib.load(open('samconfig.toml','rb')); \
                            print(cfg[e]['deploy']['parameters']['region'])"
                        """,
                        returnStdout: true
                    ).trim()

                    env.STACK_NAME = sh(
                        script: """
                            python3 -c "import tomllib,os; e=os.environ.get('CONFIG_ENV','production'); \
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
                        --disable-warnings \
                        -m "api and readonly"
                '''
            }
        }
    }

    post {
        success { echo 'Pipeline CD completado con éxito.' }
        failure { echo 'Pipeline CD falló.' }
        always {
            // Limpieza (evita basura en el workspace)
            sh 'rm -f samconfig.toml || true'
            sh 'rm -rf config-repo || true'
        }
    }
}
