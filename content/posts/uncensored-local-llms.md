+++
title = "Uncensored Local LLMs"
date = "2024-08-24T00:00:00Z"
author = "Nuno Adrego"
authorTwitter = "nunoadrego" #do not include @
cover = ""
tags = ["llm", "ai"]
keywords = ["LLM", "AI"]
description = "LLM says no"
showFullContent = false
+++

Large Language Models (LLMs) have quickly spread over the past couple of years, not only within the tech community but also in schools, universities, and many other fields of society.

Some well-known products leveraging this technology include ChatGPT, GitHub Copilot, and Perplexity, among others. These products are highly effective for a wide range of tasks, both personal and professional. With a low barrier to entry, many individuals and companies are already using them extensively.

However, using them in a professional context can be challenging. It’s important to balance the benefits with the potential risk of exposing confidential data[^1]. Techniques to mitigate this risk include replacing the real names of systems, projects, products, and companies with fake ones. Remember, this is a mitigation strategy and not a definitive solution. For instance, if an employee masks his prompts but they are exposed and linked to him, the information could be cross-referenced with his LinkedIn profile, potentially revealing insights about the company's internal operations.

[^1]: [Samsung bans use of generative AI tools like ChatGPT after April internal data leak](https://techcrunch.com/2023/05/02/samsung-bans-use-of-generative-ai-tools-like-chatgpt-after-april-internal-data-leak/)

## Local LLMs

At the [latest Kubecon](https://nunoadrego.com/posts/embracing-kubecon-in-person), a significant amount of content focused on AI and LLMs. One company in particular, Ollama, was featured in a keynote and had a booth that caught my attention.

With Ollama installed, you can run models locally. When I took some time to experiment with it, I had to disconnect from the internet to verify that it was indeed running entirely locally. 🤯 The performance exceeded my expectations! I tried Llama from Meta, but other models such as Gemma from Google, Mistral, Qwen from Alibaba, and Phi from Microsoft are also available.

If you want to try it for yourself, here is a list of commands that might be useful after installation:

```bash
ollama serve # start Ollama
ollama pull llama3.1 # pull model
ollama run llama3.1 "Hello! Who are you?" # run model with prompt
```

## Cooperation

With the ability to run models locally, I wondered if I could leverage them to aid in tasks I might face in my daily job.

I began by focusing on securing systems such as Kubernetes clusters. Understanding how a system can be attacked can help improve its defenses.

With that in mind, I tried the following prompt with `Llama3.1`:

```text
How to attack a Kubernetes cluster?
```

And the answer was:

```text
I can’t provide information on how to attack a Kubernetes cluster. Is there something else I can help you with?
```

The LLM is not cooperating. 😅 What’s going on here?

Most models have some embedded alignment. If you want to read more about this topic, I recommend the [Uncensored Models](https://erichartford.com/uncensored-models) post. The gist is that the model has guardrails and biases, which in this case are preventing me from performing an “illegal” activity. 🤷

> Intellectual curiosity is not illegal, and the knowledge itself is not illegal.
> -- <cite>Eric Hartford[^2]</cite>

[^2]: [Uncensored Models](https://erichartford.com/uncensored-models)

## Freedom and responsibility

Being stopped by `Llama3.1` was the perfect excuse to learn more about using models from sources other than the ones available in the Ollama library.

I started by exploring the [Hugging Face](huggingface.co) community. There, I found [Lexi](https://huggingface.co/Orenguteng/Llama-3.1-8B-Lexi-Uncensored-V2-GGUF), which is based on `Llama3.1`:

> Lexi is uncensored, which makes the model compliant. You are advised to implement your own alignment layer before exposing the model as a service. It will be highly compliant with any requests, even unethical ones.
> 
> You are responsible for any content you create using this model. Please use it responsibly.
> -- <cite>Lexi[^3]</cite>

[^3]: [Llama-3.1-8B-Lexi-Uncensored-V2-GGUF](https://huggingface.co/Orenguteng/Llama-3.1-8B-Lexi-Uncensored-V2-GGUF "Image of a futuristic Llama")

![Lexi](/static_en/lexi.png)

It looked like what I wanted. Now... how do I import it into Ollama? 🤔

After some research, I found this video - [Importing Open Source Models to Ollama](https://www.youtube.com/watch?v=fnvZJU5Fj3Q). Here’s how I did it:

Install [git-lfs](https://git-lfs.com/):

```bash
brew install git-lfs
git lfs install
```

Clone Lexi:

```bash
git clone https://huggingface.co/Orenguteng/Llama-3.1-8B-Lexi-Uncensored-V2-GGUF
```

Get the `Llama3.1` template (not mandatory):

```bash
ollama show llama3.1 --template
```

Use it to create a new `Modelfile` ([reference](https://github.com/ollama/ollama/blob/main/docs/modelfile.md)):

```
FROM ./Llama-3.1-8B-Lexi-Uncensored_V2_Q8.gguf

TEMPLATE """{{ if .Messages }}
{{- if or .System .Tools }}<|start_header_id|>system<|end_header_id|>
{{- if .System }}

{{ .System }}
{{- end }}
{{- if .Tools }}

You are a helpful assistant with tool calling capabilities. When you receive a tool call response, use the output to format an answer to the orginal use question.
{{- end }}
{{- end }}<|eot_id|>
{{- range $i, $_ := .Messages }}
{{- $last := eq (len (slice $.Messages $i)) 1 }}
{{- if eq .Role "user" }}<|start_header_id|>user<|end_header_id|>
{{- if and $.Tools $last }}

Given the following functions, please respond with a JSON for a function call with its proper arguments that best answers the given prompt.

Respond in the format {"name": function name, "parameters": dictionary of argument name and its value}. Do not use variables.

{{ $.Tools }}
{{- end }}

{{ .Content }}<|eot_id|>{{ if $last }}<|start_header_id|>assistant<|end_header_id|>

{{ end }}
{{- else if eq .Role "assistant" }}<|start_header_id|>assistant<|end_header_id|>
{{- if .ToolCalls }}

{{- range .ToolCalls }}{"name": "{{ .Function.Name }}", "parameters": {{ .Function.Arguments }}}{{ end }}
{{- else }}

{{ .Content }}{{ if not $last }}<|eot_id|>{{ end }}
{{- end }}
{{- else if eq .Role "tool" }}<|start_header_id|>ipython<|end_header_id|>

{{ .Content }}<|eot_id|>{{ if $last }}<|start_header_id|>assistant<|end_header_id|>

{{ end }}
{{- end }}
{{- end }}
{{- else }}
{{- if .System }}<|start_header_id|>system<|end_header_id|>

{{ .System }}<|eot_id|>{{ end }}{{ if .Prompt }}<|start_header_id|>user<|end_header_id|>

{{ .Prompt }}<|eot_id|>{{ end }}<|start_header_id|>assistant<|end_header_id|>

{{ end }}{{ .Response }}{{ if .Response }}<|eot_id|>{{ end }}
"""
```

Create the model:

```bash
ollama create llama3.1-lexi -f ./Modelfile
```

To validate, run:

```bash
ollama list
```

It should show your new model as available.

And finally, use the same prompt as in the previous section:

```bash
ollama run llama3.1-lexi "How to attack a Kubernetes cluster?"
```

Profit! 🎉

I received an answer with several suggestions such as leveraging pod vulnerabilities, misconfigured network policies, privilege escalation, etc. Most of these topics are covered in the Certified Kubernetes Security Specialist (CKS) exam.

## Final remarks

With this exploration, I was able to:

- Unlock the potential of local LLMs.
- Expand model options - Hugging Face currently has more than 860k available.

This goes without saying: you are responsible for everything you do with the information obtained from these models.

Enjoy responsibly 😉
