pipeline {
    agent any
    
    parameters {
        string(name: 'MODEL_NAME', description: 'Hugging Face model name (e.g., microsoft/phi-2)', defaultValue: '')
        string(name: 'REVISION', description: 'Model revision (branch or tag)', defaultValue: 'main')
        choice(name: 'DOWNLOAD_STRATEGY', 
            choices: [
                'all', 
                'model_files_only', 
                'weights_only', 
                'config_only', 
                'tokenizer_only'
            ], 
            description: 'Select which files to download')
    }
    
    environment {
        HUGGINGFACE_URL = "https://huggingface.co"
        MINIO_URL = "http://localhost:9000"
        BUCKET_NAME = "models"
        // Ensure credentials are set up in Jenkins
        HUGGINGFACE_API_TOKEN = credentials('huggingface-token')
    }
    
    stages {
        stage('Validate Input') {
            steps {
                script {
                    if (!params.MODEL_NAME) {
                        error "Model name is required. Please provide a valid Hugging Face model name."
                    }
                    
                    echo "🔍 Preparing to download model: ${params.MODEL_NAME}"
                    echo "📦 Revision: ${params.REVISION}"
                    echo "🗂️ Download Strategy: ${params.DOWNLOAD_STRATEGY}"
                }
            }
        }
        
        stage('Prepare Download Environment') {
            steps {
                script {
                    def modelPath = "models/${params.MODEL_NAME}"
                    sh """
                        mkdir -p ${modelPath}
                        # Ensure directory is clean
                        rm -rf ${modelPath}/*
                    """
                }
            }
        }
        
        stage('Fetch Model File List') {
            steps {
                script {
                    def modelPath = "models/${params.MODEL_NAME}"
                    
                    // Fetch model files from Hugging Face API
                    def filesResponse = httpRequest(
                        url: "${env.HUGGINGFACE_URL}/api/models/${params.MODEL_NAME}/tree/${params.REVISION}",
                        httpMode: 'GET',
                        customHeaders: [[name: 'Authorization', value: "Bearer ${env.HUGGINGFACE_API_TOKEN}"]],
                        consoleLogResponseBody: false,
                        validResponseCodes: '200',
                        quiet: true
                    )
                    
                    def files = []
                    def jsonText = filesResponse.content
                    
                    // Advanced file filtering based on download strategy
                    def fileFilters = [
                        'all': { file -> true },
                        'model_files_only': { file -> 
                            file ==~ /.*\.(bin|pt|pth|ckpt|safetensors|model|h5)$/ 
                        },
                        'weights_only': { file -> 
                            file ==~ /.*\.(bin|pt|pth|ckpt|safetensors|weights)$/ 
                        },
                        'config_only': { file -> 
                            file ==~ /config\.json|model\.safetensors/ 
                        },
                        'tokenizer_only': { file -> 
                            file ==~ /tokenizer\.json|tokenizer_config\.json|vocab\.json/ 
                        }
                    ]
                    
                    def filePattern = ~/"path":"([^"]+)","type":"file"/
                    def matcher = filePattern.matcher(jsonText)
                    
                    while (matcher.find()) {
                        def file = matcher.group(1)
                        def filterFunc = fileFilters[params.DOWNLOAD_STRATEGY]
                        
                        if (file != '.gitattributes' && filterFunc(file)) {
                            files.add(file)
                        }
                    }
                    
                    if (files.isEmpty()) {
                        error "No matching files found for model ${params.MODEL_NAME} with strategy ${params.DOWNLOAD_STRATEGY}"
                    }
                    
                    // Store files for next stage
                    env.MODEL_FILES = files.join(',')
                    echo "📥 Files to download: ${env.MODEL_FILES}"
                }
            }
        }
        
        stage('Download Model Files') {
            steps {
                script {
                    def modelPath = "models/${params.MODEL_NAME}"
                    def files = env.MODEL_FILES.split(',')
                    
                    files.each { file ->
                        def url = "${env.HUGGINGFACE_URL}/${params.MODEL_NAME}/resolve/${params.REVISION}/${file}"
                        def outputPath = "${modelPath}/${file}"
                        
                        // Robust download with multiple methods
                        sh """
                            set +e
                            
                            # Try wget first
                            wget -q --tries=3 --timeout=300 \
                                --header="Authorization: Bearer ${env.HUGGINGFACE_API_TOKEN}" \
                                "${url}" -O "${outputPath}"
                            
                            # If wget fails, try curl
                            if [ $? -ne 0 ]; then
                                curl -sSL -m 300 \
                                    -H "Authorization: Bearer ${env.HUGGINGFACE_API_TOKEN}" \
                                    "${url}" -o "${outputPath}"
                            fi
                            
                            # Verify download
                            if [ ! -s "${outputPath}" ]; then
                                echo "❌ Failed to download ${file}"
                                exit 1
                            fi
                        """
                        
                        echo "✅ Downloaded: ${file}"
                    }
                }
            }
        }
        
        stage('Store Model in MinIO') {
            steps {
                script {
                    def modelPath = "models/${params.MODEL_NAME}"
                    
                    // Verify files were downloaded
                    def modelFiles = sh(
                        script: "ls -A ${modelPath} | wc -l", 
                        returnStdout: true
                    ).trim()
                    
                    if (modelFiles.toInteger() == 0) {
                        error "❌ Model download directory is empty!"
                    }
                    
                    // Upload to MinIO with credentials
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'minio-credentials', 
                            usernameVariable: 'MINIO_USER', 
                            passwordVariable: 'MINIO_PASS'
                        )
                    ]) {
                        sh """
                            # Configure MinIO client
                            /usr/local/bin/mc alias set myminio ${MINIO_URL} ${MINIO_USER} ${MINIO_PASS} --quiet || true
                            
                            # Create bucket if not exists
                            if ! /usr/local/bin/mc ls myminio/${BUCKET_NAME} >/dev/null 2>&1; then
                                /usr/local/bin/mc mb myminio/${BUCKET_NAME}
                            fi
                            
                            # Copy model files
                            /usr/local/bin/mc cp --recursive ${modelPath} myminio/${BUCKET_NAME}/
                        """
                    }
                    
                    echo "✅ Model successfully stored in MinIO"
                }
            }
        }
    }
    
    post {
        success {
            echo "🎉 Model download and storage completed successfully!"
        }
        failure {
            echo "❌ Model download or storage failed. Check logs for details."
        }
        cleanup {
            script {
                // Optional: Clean up local model files
                sh "rm -rf models/${params.MODEL_NAME}"
            }
        }
    }
}
