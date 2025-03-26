# llm-download-from-huggingface
A simple parameterized pipeline that downloads an LLM depending on its type
Instructions:
Build Jenkins container first
'''bash
docker build -t jenkins .
'''
The run docker-compose file
'''bash
docker-compose up -d
'''
Access Jenkins at localhost:8080 and Minio at localhost:9000 and play with it
I DID not use any vaults to hide the passwords as it is only for testing purposes. 
