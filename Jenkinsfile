stage('Download Model') {
    steps {
        script {
            // Enhanced logging and error checking for model downloads
            def MIN_MODEL_FILE_SIZE = 1024 * 1024 * 1024 // 1 GB in bytes
            def modelFiles = []
            
            // Validate input parameters
            if (!env.MODEL_NAME) {
                error "MODEL_NAME must be specified"
            }
            if (!env.MODEL_TYPE) {
                error "MODEL_TYPE must be specified"
            }
            
            // Determine model files based on model type with more validation
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

            // Create model directory with full path
            sh "mkdir -p models/${env.MODEL_NAME}"

            // Comprehensive download and validation for each file
            for (file in modelFiles) {
                def url = "${HUGGINGFACE_URL}/${env.MODEL_NAME}/resolve/main/${file}"
                def outputPath = "models/${env.MODEL_NAME}/${file}"

                // Verbose download script with multiple download methods and logging
                def downloadScript = """
                    echo "Attempting to download: ${file}"
                    echo "URL: ${url}"
                    
                    # Attempt download with wget
                    wget --header="Authorization: Bearer $HUGGINGFACE_API_TOKEN" \
                         --content-disposition \
                         --continue \
                         --max-redirect=10 \
                         --verbose \
                         -O ${outputPath} "${url}" || true

                    # If wget fails, try curl
                    if [ ! -s "${outputPath}" ]; then
                        echo "wget failed, trying curl"
                        curl -H "Authorization: Bearer $HUGGINGFACE_API_TOKEN" \
                             -sSL \
                             --max-time 3600 \
                             --retry 3 \
                             --retry-delay 10 \
                             -o ${outputPath} "${url}" || true
                    fi

                    # Check file size and integrity
                    FILE_SIZE=$(stat -c%s "${outputPath}")
                    echo "Downloaded file size: $FILE_SIZE bytes"

                    if [ $FILE_SIZE -lt ${MIN_MODEL_FILE_SIZE} ]; then
                        echo "ERROR: File too small (${file})"
                        exit 1
                    fi
                """

                def exitCode = sh(
                    script: downloadScript,
                    returnStatus: true
                )

                if (exitCode != 0) {
                    error "Failed to download or validate ${file}. Check logs for details."
                }
            }

            // Optional: Additional model validation
            sh """
                echo "Model download complete. Contents:"
                ls -lh models/${env.MODEL_NAME}
            """
        }
    }
}
