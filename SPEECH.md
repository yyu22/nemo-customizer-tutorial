# 5-Minute Technical Walkthrough: Fine-Tuning Nemotron-Super with NeMo Customizer

## Introduction (30 seconds)

Hey everyone! So today I want to show you how ridiculously easy it's become to fine-tune a 49 billion parameter model. We're talking about NVIDIA's Llama-3.3-Nemotron-Super using NeMo Customizer. If you've done distributed training before, you know it's usually a nightmare of config files and infrastructure. But this? It's basically just API calls. Let me walk you through the whole stack.

## Step 0: Environment Setup (1 minute)

Alright, first things first—we need to get the environment running. Three main pieces here.

We start by grabbing our API keys for NGC and Hugging Face. Pretty standard stuff—using `getpass` so we're not hardcoding credentials. NGC gets us access to NVIDIA's model registry and container images, while the HF token is for downloading base models.

Now here's the interesting part—we spin up the entire NeMo Microservices Platform locally. The deployment script is basically doing a lot of heavy lifting: it fires up Minikube, pulls down the Helm charts, and deploys three core microservices—the NeMo Customizer service itself, a NIM for inference, and the NeMo Data Store. This data store is actually pretty clever—it's a local file storage service that exposes HF-compatible APIs, so you can use the `HfApi` client you already know. The whole deployment takes maybe 10-15 minutes, and you're watching it go through the Kubernetes orchestration steps.

Finally, we install the Python SDK—`nemo-microservices` and `huggingface-hub`. These give us clean Python bindings to talk to all these services. Nothing fancy, just standard REST API wrappers.

## Step 1: Client Initialization (30 seconds)

So once everything's deployed, we initialize the NeMo client. Super straightforward—we're pointing it at three local endpoints: `nemo.test`, `nim.test`, and `data-store.test`. These are DNS entries the deployment script configured for us. The client is basically a unified interface to all the microservices—customization jobs, inference calls, deployment management, the whole stack.

## Step 2: Data Upload and Registration (1 minute)

Okay, data time. We've got three JSONL files—train, validation, and test splits. Nothing exotic.

Now here's where it gets interesting. We're using the `HfApi` client, but—and this is important—we're NOT uploading to public Hugging Face. Check out the endpoint configuration: we're pointing it at `data-store.test/v1/hf`. This is NVIDIA's NeMo Data Store running locally in our NMP environment. It just happens to speak the Hugging Face Hub protocol, which is brilliant because it means we can reuse all the HF tooling. So we're creating a namespace "nemotron-tutorial", a dataset repo, and uploading our files. It's all staying in our local environment.

Then we register this dataset with NeMo's entity store. This is a separate step that creates metadata—namespace, project association, description. The key thing is the `files_url` using that `hf://datasets` URI format. Even though it looks like a Hugging Face URL, it's actually resolving to our local data store. This metadata makes the dataset discoverable by the customization service when we kick off training jobs.

## Step 3: Running Fine-Tuning (1 minute 15 seconds)

Alright, now we actually fine-tune this thing. And honestly? It's almost anticlimactic how easy this is.

First, we query for available configs. We get back "nvidia/nemotron-super-llama-3.3-49b@v1.5+A100"—this is basically a pre-baked training recipe. It's got all the parallelism strategies, memory optimizations, and distributed training configs already figured out for A100s. We see it supports SFT with LoRA on 4 GPUs. All that NCCL, tensor parallelism, pipeline parallelism stuff? Already handled.

Creating the job is literally just a Python dict. We're doing supervised fine-tuning with LoRA—Low-Rank Adaptation for you parameter-efficient fine-tuning folks. Adapter dimension of 8, so we're only training about 0.02% of the parameters. One epoch, batch size 16, learning rate 1e-4. Pretty conservative hyperparameters, but you can tune these based on your data.

We point it at our dataset using the namespace and name from earlier, and specify an output model ID. Hit create, and boom—NeMo is scheduling pods, downloading model weights, sharding across GPUs, and running the training loop. From our side? It's just polling a status endpoint every 5 seconds. The actual training runs for however long it takes—could be minutes to hours depending on your dataset size.

## Step 4: Inference with Base and Fine-Tuned Models (1 minute 15 seconds)

Training's done, now let's actually use these models.

First, we need to deploy a NIM—NVIDIA Inference Microservice. We're spinning up the base Nemotron model on 4 GPUs. We specify the container image from NGC, allocate a 200GB persistent volume for model weights, and pass some environment variables—in this case, enabling the Outlines backend for guided decoding. This deployment pulls the container, loads the model into VRAM, and spins up the inference server. Takes 10-20 minutes for these large models.

Once it's up, we list available models and see two: the base model and our LoRA adapter. The cool thing about LoRA is it can be served alongside the base model—the adapter weights are tiny, so you can hot-swap them.

Let's hit both endpoints. First, the base model—we ask it "How many 'r's are in strawberry?" and it breaks it down letter by letter. Pretty solid reasoning. Notice we're using the `/no_think` system prompt to disable chain-of-thought, and we get a structured response counting three r's.

Now the fine-tuned model. We ask about medieval institutional structures, and it's giving us this comprehensive breakdown with examples, descriptions, formatted nicely with headers. Clearly our training data had some historical content, and the model's been specialized for this kind of knowledge retrieval and structured output.

The API is dead simple—standard OpenAI-compatible chat completions interface. Model ID, messages array, temp, max tokens. Whether you're calling the base model or your LoRA, same interface. Makes it super easy to A/B test or swap models.

## Conclusion (30 seconds)

And that's the whole stack! We went from nothing to a fine-tuned 49B model with inference endpoints in what, like 30 lines of actual Python code? 

The key insight here is that NeMo Customizer is abstracting away all the hard distributed systems problems. You're not writing SLURM scripts, you're not debugging NCCL hangs, you're not tuning parallelism configs. It's all managed services with clean APIs. But you still get full control over the important stuff—your data, your hyperparameters, your model outputs.

If you're doing any kind of LLM customization work, definitely check this out. The notebook's all there, and the local deployment makes it easy to experiment. Alright, that's it—thanks everyone!

---

**Total word count: ~750 words (approximately 5 minutes at 145-150 words per minute)**

