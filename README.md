# LLM Download from Hugging Face

## Overview
A parameterized Jenkins pipeline for downloading Large Language Models (LLMs) from Hugging Face and storing them in MinIO for easy access and management.

## Prerequisites
- Docker
- Docker Compose
- Hugging Face account (optional, but recommended for private models)

## Security Considerations
🚨 **IMPORTANT**: This is a testing/development setup. For production:
- Use secure credential management 
- Implement proper secrets handling
- Use environment-specific configurations

## Getting Started

### 1. Clone the Repository
```bash
git clone https://github.com/your-org/llm-download-from-huggingface.git
cd llm-download-from-huggingface
```

### 2. Environment Setup

#### Credentials
1. Create a `.env` file (DO NOT commit this to version control)
```
HUGGINGFACE_TOKEN=your_huggingface_token
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
```

2. Add `.env` to `.gitignore`
```bash
echo ".env" >> .gitignore
```

### 3. Build and Run

#### Build Jenkins Container
```bash
docker build -t custom-jenkins .
```

#### Start Services
```bash
docker-compose up -d
```

### 4. Access Services
- Jenkins: `http://localhost:8080`
- MinIO: `http://localhost:9000`

## Pipeline Usage

### Model Download Parameters
- `MODEL_NAME`: Full Hugging Face model path (e.g., 'gpt2', 'bert-base-uncased')
- `MODEL_TYPE`: Model framework ('pytorch', 'tensorflow', 'onnx')

### Example Pipeline Configurations
```groovy
// GPT-2 Model
params.MODEL_NAME = 'gpt2'
params.MODEL_TYPE = 'pytorch'

// BERT Model
params.MODEL_NAME = 'bert-base-uncased'
params.MODEL_TYPE = 'pytorch'
```

## Troubleshooting
- Ensure Docker and Docker Compose are installed
- Check container logs for detailed error messages
- Verify Hugging Face token permissions

## Contributing
1. Fork the repository
2. Create your feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

## License
[Choose an appropriate license, e.g., MIT]

## Disclaimer
This is a development/testing tool. Ensure compliance with Hugging Face and model licensing terms.
```
