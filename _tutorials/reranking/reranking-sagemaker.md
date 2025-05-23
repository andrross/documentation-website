---
layout: default
title: Reranking search results using a reranker in Amazon SageMaker
parent: Reranking search results
nav_order: 115
redirect_from:
  - /vector-search/tutorials/reranking/reranking-sagemaker/
---

# Reranking search results using a reranker in Amazon SageMaker

A [reranking pipeline]({{site.url}}{{site.baseurl}}/search-plugins/search-relevance/reranking-search-results/) can rerank search results, providing a relevance score for each document in the search results with respect to the search query. The relevance score is calculated by a reranker model. 

This tutorial shows you how to rerank search results in self-managed OpenSearch and [Amazon OpenSearch Service](https://docs.aws.amazon.com/opensearch-service/). The tutorial uses the [Hugging Face BAAI/bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3) model hosted on Amazon SageMaker.

Replace the placeholders beginning with the prefix `your_` with your own values.
{: .note}

## Prerequisite: Deploy the model to Amazon SageMaker

Use the following code to deploy the model to Amazon SageMaker. We suggest using a GPU for better performance:  

```python
import json
import sagemaker
import boto3
from sagemaker.huggingface import HuggingFaceModel, get_huggingface_llm_image_uri
from sagemaker.serverless import ServerlessInferenceConfig

try:
	role = sagemaker.get_execution_role()
except ValueError:
	iam = boto3.client('iam')
	role = iam.get_role(RoleName='sagemaker_execution_role')['Role']['Arn']

# Hub Model configuration. https://huggingface.co/models
hub = {
	'HF_MODEL_ID':'BAAI/bge-reranker-v2-m3'
}

# create Hugging Face Model Class
huggingface_model = HuggingFaceModel(
	image_uri=get_huggingface_llm_image_uri("huggingface-tei",version="1.2.3"),
	env=hub,
	role=role, 
)

# deploy model to SageMaker Inference
predictor = huggingface_model.deploy(
	initial_instance_count=1,
	instance_type="ml.g5.2xlarge",
  )
```
{% include copy.html %}

For more information, see [How to deploy this model using Amazon SageMaker](https://huggingface.co/BAAI/bge-reranker-v2-m3?sagemaker_deploy=true).

To perform a reranking test, use the following code:

```python
result = predictor.predict(data={
    "query":"What is the capital city of America?",
    "texts":[
        "Carson City is the capital city of the American state of Nevada.",
        "The Commonwealth of the Northern Mariana Islands is a group of islands in the Pacific Ocean. Its capital is Saipan.",
        "Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district.",
        "Capital punishment (the death penalty) has existed in the United States since beforethe United States was a country. As of 2017, capital punishment is legal in 30 of the 50 states."
    ]
})

print(json.dumps(result, indent=2))
```
{% include copy.html %}

The response contains the reranked results ordered by relevance score:

```json
[
  {
    "index": 2,
    "score": 0.92879725
  },
  {
    "index": 0,
    "score": 0.013636836
  },
  {
    "index": 1,
    "score": 0.000593021
  },
  {
    "index": 3,
    "score": 0.00012148176
  }
]
```

To sort the results by index, use the following code:

```python
print(json.dumps(sorted(result, key=lambda x: x['index']),indent=2))
```
{% include copy.html %}

The sorted results are as follows:

```json
[
  {
    "index": 0,
    "score": 0.013636836
  },
  {
    "index": 1,
    "score": 0.000593021
  },
  {
    "index": 2,
    "score": 0.92879725
  },
  {
    "index": 3,
    "score": 0.00012148176
  }
]
```

Note the model inference endpoint; you'll use it to create a connector in the next step. You can confirm the inference endpoint URL using the following code:

```python
region_name = boto3.Session().region_name
endpoint_name = predictor.endpoint_name
endpoint_url = f"https://runtime.sagemaker.{region_name}.amazonaws.com/endpoints/{endpoint_name}/invocations"
print(endpoint_url)
```
{% include copy.html %}

## Step 1: Create a connector and register the model

To create a connector for the model, send the following request. 

If you are using self-managed OpenSearch, supply your AWS credentials:

```json
POST /_plugins/_ml/connectors/_create
{
  "name": "Sagemakre cross-encoder model",
  "description": "Test connector for Sagemaker cross-encoder model",
  "version": 1,
  "protocol": "aws_sigv4",
  "credential": {
    "access_key": "your_access_key",
    "secret_key": "your_secret_key",
    "session_token": "your_session_token"
  },
  "parameters": {
    "region": "your_sagemaker_model_region_like_us-west-2",
    "service_name": "sagemaker"
  },
  "actions": [
    {
      "action_type": "predict",
      "method": "POST",
      "url": "your_sagemaker_model_inference_endpoint_created_in_last_step",
      "headers": {
        "content-type": "application/json"
      },
      "pre_process_function": """
        def query_text = params.query_text;
        def text_docs = params.text_docs;
        def textDocsBuilder = new StringBuilder('[');
        for (int i=0; i<text_docs.length; i++) {
          textDocsBuilder.append('"');
          textDocsBuilder.append(text_docs[i]);
          textDocsBuilder.append('"');
          if (i<text_docs.length - 1) {
            textDocsBuilder.append(',');
          }
        }
        textDocsBuilder.append(']');
        def parameters = '{ "query": "' + query_text + '",  "texts": ' + textDocsBuilder.toString() + ' }';
        return  '{"parameters": ' + parameters + '}';
      """,
      "request_body": """
        { 
          "query": "${parameters.query}",
          "texts": ${parameters.texts}
        }
      """,
      "post_process_function": """
        if (params.result == null || params.result.length == 0) {
          throw new IllegalArgumentException("Post process function input is empty.");
        }
        def outputs = params.result;
        def scores = new Double[outputs.length];
        for (int i=0; i<outputs.length; i++) {
          def index = new BigDecimal(outputs[i].index.toString()).intValue();
          scores[index] = outputs[i].score;
        }
        def resultBuilder = new StringBuilder('[');
        for (int i=0; i<scores.length; i++) {
          resultBuilder.append(' {"name": "similarity", "data_type": "FLOAT32", "shape": [1],');
          resultBuilder.append('"data": [');
          resultBuilder.append(scores[i]);
          resultBuilder.append(']}');
          if (i<outputs.length - 1) {
            resultBuilder.append(',');
          }
        }
        resultBuilder.append(']');
        return resultBuilder.toString();
      """
    }
  ]
}
```
{% include copy-curl.html %}

If you are using Amazon OpenSearch service, you can provide an AWS Identity and Access Management (IAM) role Amazon Resource Name (ARN) that allows access to the SageMaker model inference endpoint:

```json
POST /_plugins/_ml/connectors/_create
{
  "name": "Sagemakre cross-encoder model",
  "description": "Test connector for Sagemaker cross-encoder model",
  "version": 1,
  "protocol": "aws_sigv4",
  "credential": {
    "roleArn": "your_role_arn_which_allows_access_to_sagemaker_model_inference_endpoint"
  },
  "parameters": {
    "region": "your_sagemkaer_model_region_like_us-west-2",
    "service_name": "sagemaker"
  },
  "actions": [
    {
      "action_type": "predict",
      "method": "POST",
      "url": "your_sagemaker_model_inference_endpoint_created_in_last_step",
      "headers": {
        "content-type": "application/json"
      },
      "pre_process_function": """
        def query_text = params.query_text;
        def text_docs = params.text_docs;
        def textDocsBuilder = new StringBuilder('[');
        for (int i=0; i<text_docs.length; i++) {
          textDocsBuilder.append('"');
          textDocsBuilder.append(text_docs[i]);
          textDocsBuilder.append('"');
          if (i<text_docs.length - 1) {
            textDocsBuilder.append(',');
          }
        }
        textDocsBuilder.append(']');
        def parameters = '{ "query": "' + query_text + '",  "texts": ' + textDocsBuilder.toString() + ' }';
        return  '{"parameters": ' + parameters + '}';
      """,
      "request_body": """
        { 
          "query": "${parameters.query}",
          "texts": ${parameters.texts}
        }
      """,
      "post_process_function": """
        if (params.result == null || params.result.length == 0) {
          throw new IllegalArgumentException("Post process function input is empty.");
        }
        def outputs = params.result;
        def scores = new Double[outputs.length];
        for (int i=0; i<outputs.length; i++) {
          def index = new BigDecimal(outputs[i].index.toString()).intValue();
          scores[index] = outputs[i].score;
        }
        def resultBuilder = new StringBuilder('[');
        for (int i=0; i<scores.length; i++) {
          resultBuilder.append(' {"name": "similarity", "data_type": "FLOAT32", "shape": [1],');
          resultBuilder.append('"data": [');
          resultBuilder.append(scores[i]);
          resultBuilder.append(']}');
          if (i<outputs.length - 1) {
            resultBuilder.append(',');
          }
        }
        resultBuilder.append(']');
        return resultBuilder.toString();
      """
    }
  ]
}
```
{% include copy-curl.html %}

For more information, see the [AWS documentation](https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ml-amazon-connector.html), [this tutorial]({{site.url}}{{site.baseurl}}/vector-search/tutorials/semantic-search/semantic-search-sagemaker/), and [the AIConnectorHelper notebook](https://github.com/opensearch-project/ml-commons/blob/2.x/docs/tutorials/aws/AIConnectorHelper.ipynb).

Use the connector ID from the response to register and deploy the model:

```json
POST /_plugins/_ml/models/_register?deploy=true
{
    "name": "Sagemaker Cross-Encoder model",
    "function_name": "remote",
    "description": "test rerank model",
    "connector_id": "your_connector_id"
}
```
{% include copy-curl.html %}

Note the model ID in the response; you'll use it in the following steps.

Test the model by using the Predict API:

```json
POST _plugins/_ml/models/your_model_id/_predict
{
  "parameters": {
    "query": "What is the capital city of America?",
    "texts": [
      "Carson City is the capital city of the American state of Nevada.",
      "The Commonwealth of the Northern Mariana Islands is a group of islands in the Pacific Ocean. Its capital is Saipan.",
      "Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district.",
      "Capital punishment (the death penalty) has existed in the United States since beforethe United States was a country. As of 2017, capital punishment is legal in 30 of the 50 states."
    ]
  }
}
```
{% include copy-curl.html %}

Alternatively, you can test the model as follows:

```json
POST _plugins/_ml/_predict/text_similarity/your_model_id
{
  "query_text": "What is the capital city of America?",
  "text_docs": [
    "Carson City is the capital city of the American state of Nevada.",
    "The Commonwealth of the Northern Mariana Islands is a group of islands in the Pacific Ocean. Its capital is Saipan.",
    "Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district.",
    "Capital punishment (the death penalty) has existed in the United States since beforethe United States was a country. As of 2017, capital punishment is legal in 30 of the 50 states."
  ]
}
```
{% include copy-curl.html %}

The connector `pre_process_function` transforms the input into the format required by the previously shown parameters.

By default, the model output has the following format:

```json
[
  {
    "index": 2,
    "score": 0.92879725
  },
  {
    "index": 0,
    "score": 0.013636836
  },
  {
    "index": 1,
    "score": 0.000593021
  },
  {
    "index": 3,
    "score": 0.00012148176
  }
]
```

The connector `post_process_function` transforms the model's output into a format that the [rerank processor]({{site.url}}{{site.baseurl}}/search-plugins/search-pipelines/rerank-processor/) can interpret and orders the results by index. This adapted format is as follows:

```json
{
  "inference_results": [
    {
      "output": [
        {
          "name": "similarity",
          "data_type": "FLOAT32",
          "shape": [
            1
          ],
          "data": [
            0.013636836
          ]
        },
        {
          "name": "similarity",
          "data_type": "FLOAT32",
          "shape": [
            1
          ],
          "data": [
            0.013636836
          ]
        },
        {
          "name": "similarity",
          "data_type": "FLOAT32",
          "shape": [
            1
          ],
          "data": [
            0.92879725
          ]
        },
        {
          "name": "similarity",
          "data_type": "FLOAT32",
          "shape": [
            1
          ],
          "data": [
            0.00012148176
          ]
        }
      ],
      "status_code": 200
    }
  ]
}
```

The response contains two `similarity` objects. For each `similarity` object, the `data` array contains a relevance score for each document with respect to the query. The `similarity` objects are provided in the order of the input documents---the first object pertains to the first document. 

## Step 2: Configure a reranking pipeline

Follow these steps to configure a reranking pipeline.

### Step 2.1: Ingest test data

Send a bulk request to ingest test data:

```json
POST _bulk
{ "index": { "_index": "my-test-data" } }
{ "passage_text" : "Carson City is the capital city of the American state of Nevada." }
{ "index": { "_index": "my-test-data" } }
{ "passage_text" : "The Commonwealth of the Northern Mariana Islands is a group of islands in the Pacific Ocean. Its capital is Saipan." }
{ "index": { "_index": "my-test-data" } }
{ "passage_text" : "Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district." }
{ "index": { "_index": "my-test-data" } }
{ "passage_text" : "Capital punishment (the death penalty) has existed in the United States since beforethe United States was a country. As of 2017, capital punishment is legal in 30 of the 50 states." }
```
{% include copy-curl.html %}

### Step 2.2: Create a reranking pipeline

Create a reranking pipeline with the cross-encoder model:

```json
PUT /_search/pipeline/rerank_pipeline_sagemaker
{
    "description": "Pipeline for reranking with Sagemaker cross-encoder model",
    "response_processors": [
        {
            "rerank": {
                "ml_opensearch": {
                    "model_id": "your_model_id_created_in_step1"
                },
                "context": {
                    "document_fields": ["passage_text"]
                }
            }
        }
    ]
}
```
{% include copy-curl.html %}

If you provide multiple field names in `document_fields`, the values of all fields are first concatenated, and then reranking is performed.
{: .note}

### Step 2.3: Test the reranking

To limit the number of returned results, you can specify the `size` parameter. For example, set `"size": 2` to return the top two documents.

First, test the query without using the reranking pipeline:

```json
POST my-test-data/_search
{
  "query": {
    "match": {
      "passage_text": "What is the capital city of America?"
    }
  },
  "highlight": {
    "pre_tags": ["<strong>"],
    "post_tags": ["</strong>"],
    "fields": {"passage_text": {}}
  },
  "_source": false,
  "fields": ["passage_text"]
}
```
{% include copy-curl.html %}

The first document in the response is `Carson City is the capital city of the American state of Nevada`, which is incorrect:

```json
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 4,
      "relation": "eq"
    },
    "max_score": 2.5045562,
    "hits": [
      {
        "_index": "my-test-data",
        "_id": "1",
        "_score": 2.5045562,
        "fields": {
          "passage_text": [
            "Carson City is the capital city of the American state of Nevada."
          ]
        },
        "highlight": {
          "passage_text": [
            "Carson <strong>City</strong> <strong>is</strong> <strong>the</strong> <strong>capital</strong> <strong>city</strong> <strong>of</strong> <strong>the</strong> American state <strong>of</strong> Nevada."
          ]
        }
      },
      {
        "_index": "my-test-data",
        "_id": "2",
        "_score": 0.5807494,
        "fields": {
          "passage_text": [
            "The Commonwealth of the Northern Mariana Islands is a group of islands in the Pacific Ocean. Its capital is Saipan."
          ]
        },
        "highlight": {
          "passage_text": [
            "<strong>The</strong> Commonwealth <strong>of</strong> <strong>the</strong> Northern Mariana Islands <strong>is</strong> a group <strong>of</strong> islands in <strong>the</strong> Pacific Ocean.",
            "Its <strong>capital</strong> <strong>is</strong> Saipan."
          ]
        }
      },
      {
        "_index": "my-test-data",
        "_id": "3",
        "_score": 0.5261191,
        "fields": {
          "passage_text": [
            "Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district."
          ]
        },
        "highlight": {
          "passage_text": [
            "(also known as simply Washington or D.C., and officially as <strong>the</strong> District <strong>of</strong> Columbia) <strong>is</strong> <strong>the</strong> <strong>capital</strong>",
            "<strong>of</strong> <strong>the</strong> United States.",
            "It <strong>is</strong> a federal district."
          ]
        }
      },
      {
        "_index": "my-test-data",
        "_id": "4",
        "_score": 0.5083029,
        "fields": {
          "passage_text": [
            "Capital punishment (the death penalty) has existed in the United States since beforethe United States was a country. As of 2017, capital punishment is legal in 30 of the 50 states."
          ]
        },
        "highlight": {
          "passage_text": [
            "<strong>Capital</strong> punishment (<strong>the</strong> death penalty) has existed in <strong>the</strong> United States since beforethe United States",
            "As <strong>of</strong> 2017, <strong>capital</strong> punishment <strong>is</strong> legal in 30 <strong>of</strong> <strong>the</strong> 50 states."
          ]
        }
      }
    ]
  }
}
```

Next, test the query using the reranking pipeline:

```json
POST my-test-data/_search?search_pipeline=rerank_pipeline_sagemaker
{
  "query": {
    "match": {
      "passage_text": "What is the capital city of America?"
    }
  },
  "ext": {
    "rerank": {
      "query_context": {
         "query_text": "What is the capital city of America?"
      }
    }
  },
  "highlight": {
    "pre_tags": ["<strong>"],
    "post_tags": ["</strong>"],
    "fields": {"passage_text": {}}
  },
  "_source": false,
  "fields": ["passage_text"]
}
```
{% include copy-curl.html %}

The first document in the response is `"Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district."`, which is correct:

```json
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 4,
      "relation": "eq"
    },
    "max_score": 0.92879725,
    "hits": [
      {
        "_index": "my-test-data",
        "_id": "3",
        "_score": 0.92879725,
        "fields": {
          "passage_text": [
            "Washington, D.C. (also known as simply Washington or D.C., and officially as the District of Columbia) is the capital of the United States. It is a federal district."
          ]
        },
        "highlight": {
          "passage_text": [
            "(also known as simply Washington or D.C., and officially as <strong>the</strong> District <strong>of</strong> Columbia) <strong>is</strong> <strong>the</strong> <strong>capital</strong>",
            "<strong>of</strong> <strong>the</strong> United States.",
            "It <strong>is</strong> a federal district."
          ]
        }
      },
      {
        "_index": "my-test-data",
        "_id": "1",
        "_score": 0.013636836,
        "fields": {
          "passage_text": [
            "Carson City is the capital city of the American state of Nevada."
          ]
        },
        "highlight": {
          "passage_text": [
            "Carson <strong>City</strong> <strong>is</strong> <strong>the</strong> <strong>capital</strong> <strong>city</strong> <strong>of</strong> <strong>the</strong> American state <strong>of</strong> Nevada."
          ]
        }
      },
      {
        "_index": "my-test-data",
        "_id": "2",
        "_score": 0.013636836,
        "fields": {
          "passage_text": [
            "The Commonwealth of the Northern Mariana Islands is a group of islands in the Pacific Ocean. Its capital is Saipan."
          ]
        },
        "highlight": {
          "passage_text": [
            "<strong>The</strong> Commonwealth <strong>of</strong> <strong>the</strong> Northern Mariana Islands <strong>is</strong> a group <strong>of</strong> islands in <strong>the</strong> Pacific Ocean.",
            "Its <strong>capital</strong> <strong>is</strong> Saipan."
          ]
        }
      },
      {
        "_index": "my-test-data",
        "_id": "4",
        "_score": 0.00012148176,
        "fields": {
          "passage_text": [
            "Capital punishment (the death penalty) has existed in the United States since beforethe United States was a country. As of 2017, capital punishment is legal in 30 of the 50 states."
          ]
        },
        "highlight": {
          "passage_text": [
            "<strong>Capital</strong> punishment (<strong>the</strong> death penalty) has existed in <strong>the</strong> United States since beforethe United States",
            "As <strong>of</strong> 2017, <strong>capital</strong> punishment <strong>is</strong> legal in 30 <strong>of</strong> <strong>the</strong> 50 states."
          ]
        }
      }
    ]
  },
  "profile": {
    "shards": []
  }
}
```

To avoid writing the query twice, use the `query_text_path` instead of `query_text`, as follows:

```json
POST my-test-data/_search?search_pipeline=rerank_pipeline_sagemaker
{
  "query": {
    "match": {
      "passage_text": "What is the capital city of America?"
    }
  },
  "ext": {
    "rerank": {
      "query_context": {
         "query_text_path": "query.match.passage_text.query"
      }
    }
  },
  "highlight": {
    "pre_tags": ["<strong>"],
    "post_tags": ["</strong>"],
    "fields": {"passage_text": {}}
  },
  "_source": false,
  "fields": ["passage_text"]
}
```
{% include copy-curl.html %}