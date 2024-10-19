# LLM Dynamic Model Demo

* A POC to prove that semantic routing and LORA adapters can work together.
* Each request is semantically evaluated against pre-configured examples and forwarded onto the correct adapter.
* Requests which fail the evaluation e.g. the router doesn't know where to route them is forwarded onto the default (Phi2) base model.


**UI(openwebui.com) -> LiteLLM + Semantic Router -> vLLM with LORA adapters**

### Deploying and Configuring vLLM 
```
oc create secret generic vllm-secrets --from-literal=HUGGING_FACE_HUB_TOKEN=hf_.....
oc apply -f vllm-pvc.yaml vllm-service.yaml vllm-route.yaml vllm-deployment.yaml 
``` 

### Downloading the models
As this is a demo the models have to be downloaded manually and uploaded to the PVC. 

### Base Model
[Phi-2](https://huggingface.co/microsoft/phi-2) from Microsoft is used as the base model.

This needs to be downloaded and stored into the **/models-cache** directory on the *vllm* pod.

The two _LORA_ adapters used are:
* [Phi2-Doctor28e](https://huggingface.co/petualang/Phi2-Doctor28e/tree/main)
* [phi-2-dcot](https://huggingface.co/haritzpuerto/phi-2-dcot)

These need to be downloaded and stored in the **/models-cache/lora/** directory on the *vllm* pod.

> [!NOTE] 
> The **lora** sub-directory will need to be created beforehand.

### Chat Prompt

A recent change in the tranformers framework results in an error being thrown if a models _chat template_ isn't present in the model configuration. To overcome this we configure vLLM to use a default chat template. However this template needs to be uploaded and stored on the PVC e.g. **/models-cache/prompt/chat.jinja**

> [!NOTE] 
> The **prompt** sub-directory will need to be created beforehand.

The chat template is stored in the *ocp_resources* directory.

```{yaml}
      containers:
      - args:
        - --model
        - microsoft/phi-2
        - --download-dir
        - /models-cache
        - --dtype
        - float16
        - --max-lora-rank
        - "64"
        - --enable-lora
        - --lora-modules
        - dcot=/models-cache/lora/phi-2-dcot/
        - doctor=/models-cache/lora/phi2-doctor28e/
        - --chat-template
        - /models-cache/prompt/chat.jinja
        - --uvicorn-log-level
        - debug
```


### Running the LiteLLM proxy
```
export BASE_API=https://vllm-xxxxxx-yyy.com/v1/
litellm --config litellm-config/pass_through_config.yaml
```

LiteLLM will start listening on **http://localhost:4000**

On startup the proxy will download the _BAAI/bge-small-en-v1.5_ embedding model used in the Semantic Router.

### Running the OpenWebUI component
```
podman run -d -p 3000:8080 -e ENABLE_OLLAMA_API=false --net=host -e ENABLE_OPENAI_API=true -e GLOBAL_LOG_LEVEL=DEBUG -e OPENAI_API_KEY=sk-123 -e OPENAI_API_BASE_URL=http://localhost:4000 -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
```

The _OpenWebUI_ will be listening on **http://localhost:8080**


## Validating the deployment
To validate the connection between the _openwebui_ and the _litellm_ proxy click on the top left and you should see _phi2_, _dcot_ and _doctor_ models listed.

To validate that the proxy is working take a look at the console logs
```
INFO:     127.0.0.1:40130 - "GET /v1/models HTTP/1.1" 200 OK
[RouteChoice(name='dcot', function_call=None, similarity_score=0.6253249230126284)]
dcot
INFO:     127.0.0.1:37520 - "POST /v1/chat/completions HTTP/1.1" 200 OK
[RouteChoice(name='doctor', function_call=None, similarity_score=0.8541370129211455), RouteChoice(name='dcot', function_call=None, similarity_score=0.7889642869682834)]
doctor
INFO:     127.0.0.1:44268 - "POST /v1/chat/completions HTTP/1.1" 200 OK
```

# Configuring the Semantic Router

The semantic router is invoked by a LiteLLM pre-invoke function and is run before the call to the actual LLM endpoint is made. This functions uses the **semantic router** framework to decide which models the request should be sent to.

The code is located in **litellm-config/custom_router.py**

# References
* https://github.com/BerriAI/litellm
* https://github.com/aurelio-labs/semantic-router/tree/main
* https://le.qun.ch/en/blog/2023/09/11/multi-lora-potentials/
* https://le.qun.ch/en/blog/2023/05/13/transformer-batching/
* https://github.com/kserve/kserve/blob/master/ROADMAP.md

Massive thanks to all these projects and the people involved. 
