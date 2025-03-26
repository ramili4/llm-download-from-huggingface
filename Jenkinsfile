pipeline {
    agent any
    parameters {
        string(name: 'MODEL_NAME', description: 'Название модели на Hugging Face')
        string(name: 'MODEL_TYPE', description: 'Тип модели (pytorch/tensorflow/onnx)', defaultValue: 'pytorch')
    }
    environment {
        MINIO_URL = "http://localhost:9000"
        BUCKET_NAME = "models"
        HUGGINGFACE_URL = "https://huggingface.co"
        HUGGINGFACE_API_TOKEN = credentials('huggingface-token')
    }
    stages {
        stage('Скачивание модели') {
            steps {
                script {
                    // Определяем файлы для скачивания в зависимости от типа модели
                    def modelFiles = [
                        'pytorch': ["pytorch_model.bin", "config.json"],
                        'tensorflow': ["tf_model.h5", "config.json"],
                        'onnx': ["model.onnx", "config.json"]
                    ]
                    
                    // Проверяем корректность типа модели
                    if (!modelFiles.containsKey(params.MODEL_TYPE)) {
                        error "Неподдерживаемый тип модели: ${params.MODEL_TYPE}"
                    }
                    
                    // Создаем директорию для модели
                    sh "mkdir -p models/${params.MODEL_NAME}"
                    
                    // Скачиваем файлы модели
                    modelFiles[params.MODEL_TYPE].each { file ->
                        def url = "${HUGGINGFACE_URL}/${params.MODEL_NAME}/resolve/main/${file}"
                        def outputPath = "models/${params.MODEL_NAME}/${file}"
                        
                        sh """
                            wget --header="Authorization: Bearer $HUGGINGFACE_API_TOKEN" -q ${url} -O ${outputPath} || \
                            curl -H "Authorization: Bearer $HUGGINGFACE_API_TOKEN" -sSL ${url} -o ${outputPath}
                        """
                    }
                }
            }
        }
        
        stage('Сохранение в MinIO') {
            steps {
                script {
                    def modelPath = "${WORKSPACE}/models/${params.MODEL_NAME}"
                    
                    // Проверяем, что модель скачалась
                    def modelFiles = sh(
                        script: "ls -A ${modelPath} | wc -l", 
                        returnStdout: true
                    ).trim()
                    
                    if (modelFiles.toInteger() == 0) {
                        error "Ошибка: Папка для модели пуста!"
                    }
                    
                    // Работа с MinIO
                    withCredentials([usernamePassword(credentialsId: 'minio-credentials', usernameVariable: 'MINIO_USER', passwordVariable: 'MINIO_PASS')]) {
                        sh """
                            /usr/local/bin/mc alias set myminio ${MINIO_URL} ${MINIO_USER} ${MINIO_PASS} --quiet
                            /usr/local/bin/mc mb -p myminio/${BUCKET_NAME}
                            /usr/local/bin/mc cp --recursive ${modelPath} myminio/${BUCKET_NAME}/
                        """
                    }
                    
                    echo "✅ Модель ${params.MODEL_NAME} успешно сохранена в MinIO"
                }
            }
        }
    }
    
    // Добавляем пост-действия для очистки
    post {
        always {
            // Очищаем временные файлы
            sh "rm -rf models/${params.MODEL_NAME}"
        }
        success {
            echo "Пайплайн завершен успешно!"
        }
        failure {
            echo "Ошибка при выполнении пайплайна"
        }
    }
}
