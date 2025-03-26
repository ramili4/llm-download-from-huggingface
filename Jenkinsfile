pipeline {
    agent any

    parameters {
        string(name: 'CONFIG_PATH', defaultValue: 'config.json', description: 'Path to config.json')
    }

    environment {
        MINIO_ALIAS = "minio"
        MINIO_BUCKET = "models"
        HUGGINGFACE_URL = "https://huggingface.co"
        HUGGINGFACE_API_TOKEN = credentials('huggingface-token')  // Загружаем токен Hugging Face
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
                    env.REVISION = config.revision ?: "main"  // По умолчанию берем "main"
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
                        sh """
                            wget --header='Authorization: Bearer $HUGGINGFACE_API_TOKEN' \
                                 -q ${HUGGINGFACE_URL}/${env.MODEL_NAME}/resolve/${env.REVISION}/${file} \
                                 -O models/${env.MODEL_NAME}/${file}
                        """
                    }
                }
            }
        }

        stage('Upload to MinIO') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'minio-creds', usernameVariable: 'MINIO_ACCESS_KEY', passwordVariable: 'MINIO_SECRET_KEY')]) {
                    script {
                        sh """
                            mc alias set ${MINIO_ALIAS} http://minio-server:9000 $MINIO_ACCESS_KEY $MINIO_SECRET_KEY
                            mc cp --recursive models/${env.MODEL_NAME} ${MINIO_ALIAS}/${MINIO_BUCKET}/
                        """
                    }
                }
            }
        }
    }
}
