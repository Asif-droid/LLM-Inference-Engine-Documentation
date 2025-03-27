# Running Ollama Locally in Docker

This guide explains how to set up and run Ollama locally using Docker, including configuring all required environment variables.

## Prerequisites

1. Install [Docker](https://www.docker.com/).
2. Clone the Ollama repository or ensure you have access to the Docker image.

## Steps to Run Ollama Locally

1. **Pull the Docker Image**  
   Pull the Ollama Docker image from the repository:
   ```bash
   docker pull ollama/ollama:latest
   ```

2. **Set Environment Variables**  
   Define the required environment variables. Below is an example of commonly used variables:
   [All env list](https://github.com/ollama/ollama/blob/d006e1e09be4d3da3fb94ab683aa18822af4b956/envconfig/config.go#L245)
   ```bash
   environment:
      - OLLAMA_FLASH_ATTENTION=1
      - OLLAMA_KEEP_ALIVE=2h
      - OLLAMA_NUM_PARALLEL=20
      - OLLAMA_MAX_LOADED_MODELS=3
   ```

3. **Run the Docker Container**  
   This is the docekr compose I used to run Ollama locally.
   - DockerFile: 
   ```
   FROM ollama/ollama:latest
   ```
   - RUN ON GPU
   ```bash
    services:

        llm_new:
            build: ./LLM
            volumes:
            - ./lama/ollama-cache/:/root/.ollama/
            - ./lama/modelfiles/:/modelfiles/
            # - ./lama/adapters/:/adapters/
            restart: always
            environment:
            - OLLAMA_FLASH_ATTENTION=1
            - OLLAMA_KEEP_ALIVE=2h
            - OLLAMA_NUM_PARALLEL=20
            - OLLAMA_MAX_LOADED_MODELS=3
            - OLLAMA_GPU_OVERHEAD=10737418240
            ports:
            - 7555:11434
            deploy:
            resources:
                reservations:
                devices:
                    - driver: nvidia
                    count: all
                    capabilities: [gpu]
   ```
    - RUN ON CPU
    ``` bash
    services:

        llm_new:
            build: ./LLM
            volumes:
            - ./lama/ollama-cache/:/root/.ollama/
            - ./lama/modelfiles/:/modelfiles/
            # - ./lama/adapters/:/adapters/
            restart: always
            environment:
            - OLLAMA_FLASH_ATTENTION=1
            - OLLAMA_KEEP_ALIVE=2h
            - OLLAMA_NUM_PARALLEL=1
            ports:
            - 7555:11434
            

    ```
   
   - The application will be accessible at `http://llm_new:11434/api/generate`.

4. **Verify the Setup**  
   Check the logs to ensure the container is running correctly:
   ```bash
   docker logs <container_id>
   ```

   Replace `<container_id>` with the ID of your running container.

5. Running Models from [Ollama.com](https://ollama.com/)
Search the model by name and if its on ollama.com. Run this command.
```bash
ollama run < model >
```
This will pull and run the model.
Pulling without running the model.
```bash
ollama pull < model >
```
6. Running Models from [huggingface.com](https://huggingface.co/)
- Download the model (any quantization available or base safetensors)
- create a modelfile like this (hermes example)
```bash
FROM < location/of/downloaded/model >
TEMPLATE """{{ if .System }}<|im_start|>system
{{ .System }}<|im_end|>{{ end }}
<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
"""

PARAMETER stop <|start_header_id|>
PARAMETER stop <|end_header_id|>
PARAMETER stop <|eot_id|>
``` 
- Run this command from inside ollama container
```bash
ollama create <new-model-name > -f <location/of/modelfile >
```

7. Running Lora adapters on base model
- Download or create the lora adapter weights.
- Mount the volume of the weights in the container.
- Write a modelfile: (Example on llama-3.1 with [lora weights](https://huggingface.co/Wengwengwhale/llama-3.1-8B-Instruct-Finance-lora-adapter))

```bash
FROM ./gguf-base/llama-3.1-8b-instruct-q4_k_m.gguf
ADAPTER ./adapters/models--Wengwengwhale--llama-3.1-8B-Instruct-Finance-lora-adapter/snapshots/c2833222686d2367a572b048acb545649c36152a
```
- Create model with 
```bash
 ollama create <new-model-name > -f <location/of/modelfile >
```

8. Api Hit
- This is the simplest format that I use:
```bash
ollama_payload = {
    "model": "model_name",
    "system": "System_info",
    "prompt": "Prompts",
    "format": format, #json schema can be defined here
    "stream": False, #full response at once
    "options": {
        "temperature": 0.3, #determines how generative 
        "num_ctx": 8192 #important. determines context len. poosible values 1024, 2048(default), 4096, 8192. If input exceeds the given num_ctx response will be empty. Setting high value will consume more memory.
    }
}
ollama_response = requests.post(url=OLLAMA_API_ENDPOINT,headers={"Content-Type": "application/json"},
data=json.dumps(ollama_payload))

Response = json.loads(ollama_response.json()['response'])

```

