pipeline {
    agent any

    parameters {
        string(name: 'CONFIG_PATH', defaultValue: 'config.json', description: 'Path to config.json')
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

        stage('Load Config') {
            steps {
                script {
                    def config = readJSON file: params.CONFIG_PATH
                    env.MODEL_NAME = config.model_name
                    env.MODEL_TYPE = config.model_type
                    env.REVISION = config.revision ?: "main"  // Default revision is "main"
                    echo "Model: ${env.MODEL_NAME}, Type: ${env.MODEL_TYPE}, Revision: ${env.REVISION}"
                }
            }
        }

    stage('Download Model') {
        steps {
            script {
                def modelFiles = []
                switch(env.MODEL_TYPE) {
                    case "pytorch":
                        modelFiles = ["pytorch_model.bin", "config.json"]
                        break
                    case "tensorflow":
                        modelFiles = ["tf_model.h5", "config.json"]
                        break
                    case "onnx":
                        modelFiles = ["model.onnx", "config.json"]
                        break
                    default:
                        error "Unsupported model type: ${env.MODEL_TYPE}"
                }
    
                sh "mkdir -p models/${env.MODEL_NAME}"
                for (file in modelFiles) {
                    def url = "${HUGGINGFACE_URL}/${env.MODEL_NAME}/resolve/main/${file}"
                    def outputPath = "models/${env.MODEL_NAME}/${file}"
    
                    // Пробуем скачать с помощью wget, если ошибка — используем curl
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
                def modelPath = "${WORKSPACE}/models/${env.MODEL_NAME}"
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
