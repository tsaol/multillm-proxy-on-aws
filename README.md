
## MultiLLM Proxy on AWS

One-Click deployment with CloudFormation

Based on LiteLLM Project [Docs](https://litellm.vercel.app/docs/)

### Outputs for API call

For deployment on ecs, use ``` http://${lb_dns}:80/v1/chat/completions ```

<img src="img/stack-ecs-elb.png" alt="Alt Text" width="800" height="">



For deployment on ec2, use ```http://${ec2_ip}:80/v1/chat/completions ```

<img src="img/stack-ec2-ip.png" alt="Alt Text" width="800" height="">


### When you update proxy config

For deployment on ecs, upload new config proxy file in s3 bucket then

```

aws ecs update-service --cluster <cluster-name> --service <service-name> --force-new-deployment

```

For deployment on ec2, login from aws console ec2 connection with session manager then

```
cd /home/ec2-user
vi proxy_config.yaml
docker-compose up -d
```

## LiteLLM Proxy Functionalities

### Simple API call 

compatible with OpenAI chat completion

### Virtual Key and Spend Tracing - with PG

#### generate virtual key 

```
curl http://$litellm_ip:80/key/generate \
    --header 'Authorization: Bearer sk-1234' \
    --header 'Content-Type: application/json' \
    --data-raw '{
        "models": [
            "bedrock-claude-v3-sonnet",
            "bedrock-claude-v3-haiku"
        ],
        "metadata": {
            "user": "chenxqdu@amazon.com"
        }
}'
```
It returns a virtual key sk-pyHgYR8Iacc4CLHwjeKu4Q.

#### request with virtual key

```
curl http://$litellm_ip:80/v1/chat/completions \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer sk-7xEL6aSZW_HkUojYS558iA" \
    -d '{
        "model": "bedrock-claude-v3-sonnet",
        "messages": [
            {
                "role": "user",
                "content": "Hello"
            }
        ]
}'
```

#### get spend info with key

```
curl http://$litellm_ip:80/key/info?key=sk-pyHgYR8Iacc4CLHwjeKu4Q \
    -X GET \
    -H 'Authorization: Bearer sk-1234'
```

### Multi LLM instance load balancing - with Redis

#### configure multi instances for a model in proxy config file
```
  - model_name: bedrock-claude-v3-sonnet
    litellm_params:
        model: bedrock/anthropic.claude-3-sonnet-20240229-v1:0
        aws_region_name: "us-west-2"
        aws_access_key_id: "xxxxxx"
        aws_secret_access_key: "xxxxxxx"
        rpm: 30
  - model_name: bedrock-claude-v3-sonnet
    litellm_params:
        model: bedrock/anthropic.claude-3-sonnet-20240229-v1:0
        aws_region_name: "us-east-1"
        aws_access_key_id: "xxxxxx"
        aws_secret_access_key: "xxxxxxx"
        rpm: 30
```

#### simple call test 

Invoke request mutliple times with -i

```

curl -i http://$litellm_ip:80/v1/chat/completions \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer sk-1234" \
    -d '{
        "model": "bedrock-claude-v3-sonnet",
        "messages": [
            {
                "role": "user",
                "content": "Hello"
            }
        ]
}'

```
Get responses and verify model-id in reposne Header 

```
HTTP/1.1 200 OK
...
server: uvicorn
x-litellm-call-id: ac125673-85ef-466e-ba3f-f26fab463041
x-litellm-model-id: 2a281c00b23a45934cb6a5cb3929249fb55627923ece219cdbcc2331a7b1855f 
...
```

```
HTTP/1.1 200 OK
...
server: uvicorn
x-litellm-call-id: c24f6b5c-385a-4105-af2f-b4ad2b801371
x-litellm-model-id: ab933985853f13d8760ccd08d89e500574fd52724a211b24e082936b8d5eb382
...
```

#### rate limit

mock rate limit 
```
curl -X POST http://$litellm_ip:80/chat/completions \
    -H 'Content-Type: application/json' \
    -H 'Authorization: Bearer sk-1234' \
    -d '{
    "model": "bedrock-claude-v3-sonnet",
    "messages": [
            {"role": "user", "content": "Hi there!"}
        ],
        "mock_testing_rate_limit_error": true
    }'
```
response error

```
{
  "error":
    {
      "message": "litellm.RateLimitError: This is a mock exception for model=bedrock-claude-v3-sonnet, to trigger a rate limit error.\nReceived Model Group=bedrock-claude-v3-sonnet\nAvailable Model Group Fallbacks=['bedrock-claude-v3-haiku']\nError doing the fallback: litellm.RateLimitError: This is a mock exception for model=bedrock-claude-v3-haiku, to trigger a rate limit error.No fallback model group found for original model_group=bedrock-claude-v3-haiku. Fallbacks=[{'bedrock-claude-v3-sonnet': ['bedrock-claude-v3-haiku']}]\nReceived Model Group=bedrock-claude-v3-haiku\nAvailable Model Group Fallbacks=None\nError doing the fallback: litellm.RateLimitError: This is a mock exception for model=bedrock-claude-v3-haiku, to trigger a rate limit error.No fallback model group found for original model_group=bedrock-claude-v3-haiku. Fallbacks=[{'bedrock-claude-v3-sonnet': ['bedrock-claude-v3-haiku']}]",
      "type": null,
      "param": null,
      "code": "429",
    },
}
```

Set rate limit for a key

```
curl --location http://$litellm_ip:80/key/generate \
    --header 'Authorization: Bearer sk-1234' \
    --header 'Content-Type: application/json' \
    --data '{"max_parallel_requests": 10, "tpm_limit": 200, "rpm_limit": 4}' 

```
Invoke request more than limited times with the returned key 

```
curl -i http://$litellm_ip:80/v1/chat/completions \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer sk-Cc1ZOD0ND6ZalAz-QjOhvw" \
    -d '{
        "model": "bedrock-claude-v3-sonnet",
        "messages": [
            {
                "role": "user",
                "content": "Hello"
            }
        ]
}'

```
And get rate limit error 

```
{
    "error": {
        "message": "Max parallel request limit reached Hit limit for api_key: 3ec122b34ca59dcefc567e561be04f3858d934b0f4168275b35b20b3875875a6. tpm_limit: 200, current_tpm 80 , rpm_limit: 4 current rpm 4,
        "type": "None",
        "param": "None",
        "code": "429"
    }
}
```

#### mock fallback

Configure fallback in proxy config file 

```
litellm_settings:
  fallbacks: [{"bedrock-claude-v3-sonnet": ["bedrock-claude-v3-haiku"]}] 
```

Request to mock fallback 

```
curl -X POST http://$litellm_ip:80/chat/completions \
    --header 'Content-Type: application/json' \
    --header 'Authorization: Bearer sk-1234' \
    --data '{
    "model": "bedrock-claude-v3-sonnet",
    "messages": [
        {
        "role": "user",
        "content": "what llm are you"
        }
    ],
    "mock_testing_fallbacks": true
}'
```
The response is returned by haiku 

```
{
    "id": "chatcmpl-c6751dcc-8d8e-4caf-86ec-4b6b051ccb7c",
    "choices": [
        {
            "finish_reason": "stop",
            "index": 0,
            "message": {
                "content": "I am an artificial intelligence language model, but I'm not entirely sure which specific large language model (LLM) I am. I was created by Anthropic, but the details of my architecture and training process are not something I have full knowledge of. I know I have been designed to engage in open-ended dialogue and assist with a variety of tasks, but the specifics of the LLM technology used to build me is not information I can provide with certainty. My focus is simply on trying to be as helpful as I can in our conversations!",
                "role": "assistant",
                "tool_calls": null,
                "function_call": null
            }
        }
    ],
    "created": 1726381662,
    "model": "anthropic.claude-3-haiku-20240307-v1:0",
    "object": "chat.completion",
    "system_fingerprint": null,
    "usage": {
        "completion_tokens": 118,
        "prompt_tokens": 12,
        "total_tokens": 130
    }
}

```

## LiteLLM Proxy UI


```
http://$litellm_ip:80/ui

Login: Admin/<master_key>
```

<img src="img/litellm_ui.png" alt="Alt Text" width="" height="">



## Reference

### Architecture for deployment on ECS

<img src="img/litellm_ecs.png" alt="Alt Text" width="500" height="">

### Architecture for deployment on EC2 

<img src="img/litellm_ec2.png" alt="Alt Text" width="500" height="">
