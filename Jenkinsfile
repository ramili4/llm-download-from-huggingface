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
                    sh(script: "mkdir -p ${modelPath}", label: 'Create model directory')

                    echo "Using Hugging Face API Token: ${env.HUGGINGFACE_API_TOKEN ? '✔️ Loaded' : '❌ Not Found'}"

                    // API-запрос для получения списка файлов модели
                    def filesResponse = httpRequest(
                        url: "${HUGGINGFACE_URL}/api/models/${params.MODEL_NAME}/tree/${params.REVISION}",
                        httpMode: 'GET',
                        customHeaders: [[name: 'Authorization', value: "Bearer ${env.HUGGINGFACE_API_TOKEN}"]],
                        consoleLogResponseBody: false,
                        quiet: true
                    )

                    def files = []
                    def jsonText = filesResponse.content
                    def filePattern = ~/"path":"([^"]+)","type":"file"/
                    def matcher = filePattern.matcher(jsonText)

                    while (matcher.find()) {
                        def file = matcher.group(1)
                        if (file != '.gitattributes') {
                            files.add(file)
                        }
                    }

                    if (files.isEmpty()) {
                        error "Не удалось найти файлы модели ${params.MODEL_NAME} с ревизией ${params.REVISION}"
                    }

                    for (file in files) {
                        def url = "${HUGGINGFACE_URL}/${params.MODEL_NAME}/resolve/${params.REVISION}/${file}"
                        def outputPath = "${modelPath}/${file}"

                        echo "Downloading: ${file}"
                        sh(
                            script: """
                                wget -q --header="Authorization: Bearer ${env.HUGGINGFACE_API_TOKEN}" "${url}" -O "${outputPath}" || \
                                curl -sSL -H "Authorization: Bearer ${env.HUGGINGFACE_API_TOKEN}" "${url}" -o "${outputPath}"
                            """,
                            label: "Download ${file}"
                        )
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
