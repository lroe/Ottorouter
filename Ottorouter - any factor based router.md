Ottorouter allows for a flexible way to route a prompt to the best LLM model by considering prompt complexity, benchmark scores, latency, and many other factors.

## What does it aim to solve?

### 1. Load Balancing
An LLM provider offers an endpoint with a fixed request rate limit, for example, 1000 requests per minute. 

A typical solution used by big companies experiencing a high number of requests per minute is to employ multiple endpoints. 

If one endpoint has reached its rate limit, the requests need to be automatically routed to another endpoint that has not reached its rate limit.



<img src="https://github.com/lroe/Ottorouter/blob/main/Pasted%20image%20240125120558.png">

### 2. Task-Complexity Based Routing
The cost per million tokens of a list of models are specified below:
```markdown
| Model          | Cost Per Million Token |
|----------------|------------------------|
|GPT 4           |  4$                    |
|GPT 3.5 Turbo   |  2.5$                  |
|Mistral         |  0.6$                  |
|Mistal          |  0.2$                  |
```

If we route every prompt to GPT-4, this results in a significant financial impact. If a developer uses 8 million tokens every day, it costs $5 when GPT-4 is utilized. Now, considering that a company has multiple developers, the cost grows exponentially.

Not every question needs to be routed to GPT-4. Some prompts, such as "What is 2+2," are basic and can be answered by more economical models like Mistal or Mistral.

This problem could be solved by assigning a complexity score to each prompt and selecting the best model based on that.


<img src="https://github.com/lroe/Ottorouter/blob/main/Pasted%20image%20240125120558.png">

Here, a complexity score is assigned to each model. For complexity above 8, prompt is routed to GPT 4.
For the rest of the complexity scores ,the routing engine uses its routing algorithm to select the best model that can efficiently respond to that prompt/


### 3. API Translation
Most LLM-based projects are built on top of the OpenAI API. Suppose we need to use another model not provided by OpenAI; in that case, we would have to modify the entire API call statements. However, if there is another way to preserve the same OpenAI API format but use a different model, it would result in a tremendous advantage.

This could be solved by the following strategy: the OpenAI API has a base environmental variable. We could change it to another local environmental variable that is handled by a routing engine so that a different LLM would answer the prompt. Additionally, when the prompt reply is obtained, it should be modified in such a way that it has the same schema as that of the OpenAI API.

This allows a user to easily try out endpoints from other providers on projects implemented with the OpenAI API.


`

````

import os
from openai import completion
from cohere import completion

os.environ["OPENAI_API_KEY"] = "openai-key"
os.environ["ANTHROPIC_API_KEY"] = "anthropic-key"

messages = [{"content": "Hello, how are you?", "role": "user"}]

# OpenAI call
response_openai = completion(model="gpt-3.5-turbo", messages=messages)

# Cohere call
response_cohere = completion(model="claude-2", messages=messages)
````


### 4. Filters
Every LLM model has a wide variety of benchmark scores like Open LLM Leaderboard score, LMSYS Arena Score, and Needle in a Haystack score.

The Open LLM Leaderboard gives a score based on answers to a specific list of questions. However, models could 'cheat' by having those questions in their training dataset.

The LMSYS Arena score is determined by providing prompt responses to humans (without specifying the model) and asking them to rate it.

The Needle in a Haystack score indicates how accurately the model can retrieve given information from a specified context length.

Similarly, there are many benchmarks that consider different characteristics of the model. Different users consider different benchmark scores.

Users should have the flexibility to specify constraints on the benchmark scores, latency,API costs  ,etc., so that a model that satisfies all the constraints is automatically selected.

In a nutshell, a query language needs to be developed so that users can easily specify the model constraints. 




### 4. Easy addition of new models

Users should be able to easily add new models to the list of available models in the router engines to route to.

To simplify this process, each model is represented as a .yaml file containing the configurations for that model.

The configuration file  for a model includes all the necessary information such as API Key, Benchmark Scores, and Latency required for the router engine to route to that model.

Adding a new model is achieved by creating a new configuration file with the required information.


<img src="https://github.com/lroe/Ottorouter/blob/main/Pasted%20image%20240125123612.png">
	a sample config file.

# Existing Solutions

### 1.  litellm, portkey, gateway :

These implement the feature to translate any LLM API call statement into OpenAI API format.


### 2.withmartin - complexity and low cost


This allows a prompt to be routed to the best LLM model by considering the complexity of the prompt. It also has a fallback feature (where  if  the chosen model is unavailable, the prompt gets routed to the next best available model.)




# Features of Ottorouter
1. Load Balancing
2. Task-Complexity Router
3. API Translation
4. Filters (A well documented **querying language** for specifying the filters!)
5. Easy addition of new models

# High-Level Overview

In-order to explain the working of Ottorouter, its best to divide it into three parts : LHS (Left Hand Side), ROUTER ENGINE, RHS (Right Hand Side)


<img src="https://github.com/lroe/Ottorouter/blob/main/Pasted%20image%20240126191925.png">

### LHS

1. Prompt : The prompt, along with the parameters like temperature is provided by the user.
2. Auto_override : If enabled, always passes the prompt to the model that can answer hard questions. For example, if we want to consider only Mixtral models, then we can used the auto_override feature to do that.
3. Filters : The benchmark constraints, latency constraint, API cost constraint etc are specified here
4. Routing strategy : *GOTTA WRITE THIS LATER!*

### RHS
1. Configured model endpoint instances: 
A model is hosted by different providers .This model provided by a provider is known as endpoint.
A customer is given access to a specific instance of an endpoint. 
This specific instance is accessed by its API Key.

The configuration file for a model endpoint instance contains all the required informations related to that model endpoint instance. This include informations like API_Key, Benchmark_Scores, Latency,API_Cost and so on.


<img src="https://github.com/lroe/Ottorouter/blob/main/Pasted%20image%20240126195433.png">


Filters should be able to filter based on any of the fields.

Why we use use files to reperesent configuration of model endpoint instances? 
-> Files are easy to handle, backup and version control.
-> We represent configuration files using YAML  which is a universal formatting language.
-> The universality approach, combined with easy configuration, gives users the freedom to contribute easily, thus enhancing the open-source nature.

# How to go about building this?
### First lets consider the Left Hand Side (LHS)


<img src="https://github.com/lroe/Ottorouter/blob/main/Pasted%20image%20240126205854.png">
1. A prompt, along with parameters like temperature is given
2. Number of tokens in that prompt is calculated. We use a specific tokenizer like llama tokenzier as a reference for all models.
3. Complexity score for that prompt is calculated.
4. Filters is already specified by the user.
5. Routing strategy takes into account the token_count, complexity score, and filters to arrive at a specific model
6. rate_limit checks if that endpoint instance has reached its rate limit or not.
7. service queue: if the endpoint instance has reached its rate limit, the response is given to a queue. From this queue, either it can be given to the endpoint when the endpoint instance becomes available or we can use fallback techniques to route to next best model endpoint instance.
# Design Issues
1. Complexity Scoring Mechanism:
		This is implemented by finetuning a cheap LLM  with examples of prompt and its complexity scores.
		
	KEY ISSUES:
		1. What if prompt length > Complexity Score Making LLM context length:
			This can be solved by using Sparse Representation Format technique.
			->A prompt can be converted into its SPF form. SPF form contains all the minimal neccessary informations so that the LLM can reconstruct the original prompt without any loss in information.
			->Note: when finetuning, in the example dataset, first convert the prompt to its corresponding SPF format. Train it using SPF format and the complexity score data.
		

2. Filter Query Language :
	 A query language should be developed to specify the filters.

### Now lets consider the RHS

<img src="https://github.com/lroe/Ottorouter/blob/main/Pasted%20image%20240126203955.png">
Here we have a list of model endpoint instance configuration files.

First we need to validate each config files.
This is done to ensure that the configuration files have the correct syntax, semantics, data etc.

However we shouldn't validate it everytime when the router engine starts. (Suppose there exists 200 config files... validating them everytime the router engine starts is just inefficient)

This problem is solved by hashing. After validating,we hash the config files.
Now, if any update is made to the config file, it will be only be reflected in the hashed file if a specific function , say 'update_hash_function' is called.

Now, we need to build a data structure to internally represent the endpoint instance informations.

## Q. How does a user add a new endpoint instance configuration file?

<img src="https://github.com/lroe/Ottorouter/blob/main/Pasted%20image%20240126210020.png">
(example directory structure of config file)
In the reference_folder, a sample configuration file containing the semantic information is there.
To add a new endpoint instance, take its corresponding sample configuration file and place it into the app.bin file with the necessary informations included in it.

# Design Issues
1. Internal Representation of endpoint instances : gotta brainstorm some solutions to this.





