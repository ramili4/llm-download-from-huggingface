pipeline {
    agent any
    parameters {
        string(name: 'MODEL_NAME', description: 'Hugging Face model name', defaultValue: 'microsoft/phi-2')
        string(name: 'REVISION', description: 'Model revision (branch or tag)', defaultValue: 'main')
    }
    environment {
        MINIO_URL = "http://localhost:9000"
        BUCKET_NAME = "models"
        HUGGINGFACE_URL = "https://huggingface.co"
        HUGGINGFACE_API_TOKEN = credentials('huggingface-token')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Download Model') {
            steps {
                script {
                    def modelPath = "models/${params.MODEL_NAME}"
                    sh "mkdir -p ${modelPath}"

                    def files = sh(
                        script: """
                            curl -sL "${HUGGINGFACE_URL}/${params.MODEL_NAME}/tree/${params.REVISION}" | \
                            grep -oP 'href="/${params.MODEL_NAME}/blob/${params.REVISION}/\K[^"]+' | \
                            grep -v '^\.gitattributes\$'
                        """,
                        returnStdout: true
                    ).trim().split('\n')

                    for (file in files) {
                        def url = "${HUGGINGFACE_URL}/${params.MODEL_NAME}/resolve/${params.REVISION}/${file}"
                        def outputPath = "${modelPath}/${file}"
                        def exitCode = sh(
                            script: """
                                wget --header="Authorization: Bearer $HUGGINGFACE_API_TOKEN" -q ${url} -O ${outputPath} || \
                                curl -H "Authorization: Bearer $HUGGINGFACE_API_TOKEN" -sSL ${url} -o ${outputPath}
                            """,
                            returnStatus: true
                        )
                        if (exitCode != 0) {
                            error "Ошибка при загрузке ${file}"
                        }
                    }
                }
            }
        }
        stage('Сохраняем модель в MinIO') {
            steps {
                script {
                    def modelPath = "${WORKSPACE}/models/${params.MODEL_NAME}"
                    def modelFiles = sh(script: "ls -A ${modelPath} | wc -l", returnStdout: true).trim()
                    if (modelFiles.toInteger() == 0) {
                        error("Ошибка: Папка для модели пуста! Выходим..")
                    }
                    withCredentials([usernamePassword(credentialsId: 'minio-credentials', usernameVariable: 'MINIO_USER', passwordVariable: 'MINIO_PASS')]) {
                        sh """
                            /usr/local/bin/mc alias set myminio ${MINIO_URL} ${MINIO_USER} ${MINIO_PASS} --quiet || true
                            if ! /usr/local/bin/mc ls myminio/${BUCKET_NAME} >/dev/null 2>&1; then
                                echo "Creating bucket ${BUCKET_NAME}..."
                                /usr/local/bin/mc mb myminio/${BUCKET_NAME}
                            fi
                            /usr/local/bin/mc cp --recursive ${modelPath} myminio/${BUCKET_NAME}/
                        """
                    }
                    echo "✅ Модель успешно сохранена в MinIO"
                }
            }
        }
    }
}
