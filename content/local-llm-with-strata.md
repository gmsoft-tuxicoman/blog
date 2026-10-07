+++
title = "Trying local LLM for agents with Strata"
date = 2026-10-07
+++

# Trying to run a local LLM again but with Strata

My last attempt to [run a local LLM with ollama](@/local-llm-with-ollama.md) on my workstation was a failure. Either the model was useless or too slow.


This week, a new backend for models showed up in hackernews: [Strata](https://github.com/Niko1221/Strata).
It made bold claims. It should be able to run 40-50 token per second on hardware similar to mine. That is a RX6800 with 16GB VRAM and 64GB of system RAM.
I had to try it!

## Installation

Installation was very easy. I went with the path of least resistance and pasted their prompt in claude:
```
Set up Strata on this PC for me: https://github.com/Niko1221/Strata - follow docs/AI_SETUP.md in that repository.
```

After a few questions, minor hiccups that claude handled well and a hefty download, it was ready.
I settled on the model qwen3.8-flash-next-iq2_xs recommended for my setup.


## How it works

Compared to ollama, Strata focuses on a single model which is a Mixture-of-Experts (MoE). Each expert can be offloaded to either the GPU or the CPU.
Strata loads the important ones and the ones you use more often in the GPU and the rest in the CPU, allowing for very fast processing when it matters.

More details can be found [here](https://github.com/Niko1221/Strata/blob/main/docs/HOW_IT_WORKS.md).

## First attempt via the Web UI

I usually test models by asking for a joke. It's an easy answer and more often than not you get the one about physicists not trusting atoms.

For once I was given a joke in no time which made me laugh. Unfortunately I did not capture it.

The model felt blazing fast! A few more queries were answered very fast as well. You can see the thinking process in the UI which is set to high by default.

Looking at the monitor page, I was getting about 30-40t/s. Not great, not terrible.

## Optimizing

Of course the first thing you want to do is get even more speed out of it!

First I used `--vram-reserve-mib 1500`. This gave me a bit of breathing room for my apps since this is my main workstation. This would not sound like an optimisation but relieving the pressure on the VRAM definitely helped.
With this change, my workstation was not hanging anymore and I consistently got around 40-50t/s.

I asked Claude for a bit of help and he pointed out that my graphic card was plugged in the wrong PCIe port. I moved it a while back to a lower 16x PCIe port because the nvme was underneath it and was getting too hot. After a lot of troubleshooting I think the issue is the sensor on the nvme and not the temperature itself. This is because PCIe ports aren't all wired the same. The top one on my motherboard is wired directly to the CPU and is a 16x one. The other one, while having a 16x slot, is only a 4x port and wired through the motherboard chipset, sharing the bandwidth with the other ports.
By moving the graphic card back to the right port, the PCIe bandwidth went from ~3Gbps to a whopping ~12Gbps. This helped a lot. I was now getting 50-60t/s!

Another small win is to enable `--draft-vocab en`. This prevents loading non-english languages and frees up a bit of VRAM allowing for more experts to load on the GPU. Gains were about 1%, not really measurable tho.


The model now felt very very fast and more than usable!


## Tuning for hermes agent

First thing I did was increase the context window. I went from 64K to 128K without losing performance. This was amazing. I haven't tried 256K yet but performance are supposed to drop by ~20% doing so.

Hermes also requests the full context size each time so you have to set `fix_max_tokens` to true.

Enabling `display.show_reasoning: true` in hermes allows you to see the model thinking as well.

One useful thing to do as well is to enable api-monitor. This gives you a great view of the queries being performed as well as their output. Use with care if you share the instance!


## Using it with hermes

Hermes now felt very useful. It managed to parse ods files, use the result to perform web searches and then give me a summary.
It was slower than Claude of course but the performances were amazing.

## The cons

There are a few gotchas you need to be aware of.

First of all, it only processes one request at a time. Each request is queued and answered one by one. This is not a big deal but a request from hermes to summarize the conversation might delay your actual work.
This might be a problem as well with sub agents or if you share access with a team.

Secondly, if you restart it, the cache is lost. The first query with a huge context will take some time to load. Subsequent ones will be faster.


## Conclusion

Strata makes local LLM a real possibility with consumer hardware. My workstation is about 5 years old and provides great results.
With my aging hardware I was able to get 50-60t/s on average.
