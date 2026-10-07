+++
title = "Trying local models for local agent using ollama"
date = 2026-08-28
+++

# Running an agent on my own hardware: some notes


I wanted to try [Hermes Agent](https://hermes-agent.ai), but I wanted to find out if I could have a fully local setup running.
The motivation was 2 fold: privacy, and cost. I'd like to keep everything local, I have nextcloud, fileserver and even my own mail server.
As for cost, if I can avoid spending $100 a month by using hardware I already have, that'd be a small win.


## The setup

As a former Gentoo dev, I'm still faithful to the distro. I'm running it on all my machines. My workstation has 64GB of RAM and a RX6800 with 16GB VRAM.

I decided to install ollama on my workstation and hermes on a test VM to limit the blast in case it would go rogue.

Ollama was fairly easy to install, `emerge ollama` did the trick after I added USE=rocm in make.conf.

## Finding the right model to run in ollama


### First attempt: qwen2.5-coder:14b

After a quick search, qwen2.5-coder:14b appeared to be a good fit for my 16GB VRAM and intended use.

Deployment is very easy with `ollama pull qwen2.5-coder:14b`, it downloads the model and allows you to use it in minutes. However when connecting Hermes to ollama, I ran into the issue that this specific model only supports 32K context length and Hermes requires at least 64K.

### Next attempt: gpt-oss:20b

That model is 13GB, not too bad to fit on my 16GB of VRAM. However, it wasn't very useful for the tasks I needed. It did work but it wasn't doing very well for tasks that required more thinking.
Sometimes, I was asking a question to the model and it was returning that question right back at me. Not very useful :)


### Another attempt: qwen3.5:9b-q8_0

That model is smaller than gpt-oss:20b, it only uses 10GB. While it does fit in the VRAM, it wasn't very useful either. 

### Trying gemma4:31b


The recommended model for hermes is gemma4:31b. Unfortunately the model you can pull with ollama is too big for my VRAM. Running it would have ollama offload half of it on the CPU.
It made things very very slow and the system became unstable.

Trying to find an alternative, Claude suggested that I could use the same model but different quantization. This would keep the same model but reduce its size.

I attempted gemma4:31b Q3_K_M, while the file on hugging face is only 14GB, it would still not fit in my VRAM:

```
NAME                 ID              SIZE     PROCESSOR          CONTEXT    UNTIL
gemma4:31b-q3_k_m    86f1b9041d07    16 GB    28%/72% CPU/GPU    64000      4 minutes from now
```

While the model is definitely more usable, it is still very slow. At least this model was able to use the MCP interface for my CRM Twenty. Previous model did not understand how to use it and were unable to list companies or contacts.


Next was gemma4:31b Q3_K_S, the file was slightly smaller but still not a win :

```
NAME                 ID              SIZE     PROCESSOR          CONTEXT    UNTIL
gemma4:31b-q3_k_s    698f500b39e8    15 GB    25%/75% CPU/GPU    64000      59 minutes from now
```


With more tweaking, using LLAMA_ARG_FIT_TARGET=0, I was able to reduce the offload to 19% but it still wasn't enough:

```
NAME                 ID              SIZE     PROCESSOR          CONTEXT    UNTIL
gemma4:31b-q3_k_s    698f500b39e8    15 GB    19%/81% CPU/GPU    64000      59 minutes from now
```

Time to reduce the quantization some more! Here comes IQ3_XXS !

```
NAME                     ID              SIZE     PROCESSOR          CONTEXT    UNTIL
gemma4:31b-ud-iq3_xxs    d99939b77c8e    13 GB    19%/81% CPU/GPU    64000      59 minutes from now
```

Despite reducing the size of the model, it still didn't fit the VRAM :-/
Even with a bit of CPU offloading, things were really slow. At least this model was still able to use the tools correctly but it was too slow to be usable for daily use.

With Claude's advice, I tried gemma4:26B, it's not a dense model but a MoE (Mixture of Experts) model. It means that the CPU offloading shouldn't hurt as much since only some part of the model will be active at once.
It indeed wasn't too slow with the request, it was responding in a matter of seconds rather than minutes.

```
NAME          ID              SIZE      PROCESSOR          CONTEXT    UNTIL
gemma4:26b    08ae7ec1744b    1.4 GB    26%/74% CPU/GPU    64000      59 minutes from now
```

Unfortunately that model didn't prove to be very useful. Even if it was faster, it still felt very slow compared to Claude and it wasn't able to use some MCP, constantly failing to issue the proper command.

## Takeaways

Unfortunately, running a local model is a tradeoff between speed and quality.
Despite many tweaks and attempts, I wasn't able to find a usable model.

A small model isn't anywhere near useful. It cannot use tools, provides very simple answers or doesn't understand the query.
Using a bigger model leads to a usable response but only if you are fine with waiting many minutes between answers. At a speed of 3-4t/s, you'll fall fast asleep.

I wanted to try this to limit the cost of my AI usage but it will slow me down more than anything else. Currently my AI use doesn't justify buying more powerful hardware.
I'll wait until the hardware price goes down or my AI usage cost goes significantly up :)

