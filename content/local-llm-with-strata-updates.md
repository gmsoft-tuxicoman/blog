+++
title = "Updates on testing local LLM with Strata"
date = 2026-10-15
+++



# Introduction


A [week ago](@/local-llm-with-strata.md) I started testing [Strata](https://github.com/Niko1221/Strata). It gave great speed compared to ollama.
However there were a few bumps on the road to get something working well with Hermes agent.
I also wanted to see if I could get more out of it. Better model for better quality or faster speed.


## The setup

 - AMD Radeon RX6800 (16GB VRAM)
 - 64GB RAM DDR4
 - AMD Ryzen 7 5700G
 - Gentoo Linux
 - Model [Qwen3.8-Flash-Next](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)


# Better models

I was running IQ2_XS but IQ3_XXS and IQ3_S are available.

## Model tweaks

In order to improve quality, a few variables were changed to match Qwen's published guidance.

| Name | Default | New value | Note |
|---|---|---|---|
| temperature | 0 (greedy) | 0.6 | Introduce a bit of randomness in the next token to avoid loops |
| top_k | 64 | 20 | Only the 20 most likely tokens can be picked | 
| top_p | 1 | 0.95 | Keep candidates until their probability adds up to 95% and drop the rest |
| presence_penalty | 0 (off)  | 1.5 | Any token that already appeared in the last 64 tokens gets a fixed penalty, however often it appeared; this is the main loop breaker |


While these changes are great to avoid loops and get better results, they introduce a slight speed penalty.


## Testing IQ3_S


Of course the first thing you do is test the best model. Well it didn't really work out well.
Although the README says that it fits, and it does, it doesn't leave enough room for essential things such as conversation cache.

The speed was 31-44t/s, so about 25% slower than the speed of IQ2_XS and even slower on longer prompts.

There is a low-RAM mode but things become even slower when I tested it. While reading prompts remained at the same speed in normal mode, it dropped to about 250-350 t/s in low-RAM mode.
This plus the fact that conversation cache did not have enough space meant that every single prompt needed to be fully re-read. It took several minutes for each request.
The low-RAM mode makes the kernel load the experts when needed instead of having dedicated RAM allocated for them.


There was a bit of quality increase in the responses. The answers were more detailed but it's all empirical and I don't really have a good way to measure this.


## Testing IQ3_XXS

This model had the promise to use less memory. And it did. Compared to the ~50GB of experts in RAM for IQ3_S, IQ3_XXS uses ~42GB in RAM. That leaves 8GB more RAM for conversation cache.


This model is about 15% faster than IQ3_S and reading speed did not change. It's about 20% slower than IQ2_XS but quality seemed identical to IQ3_S.
Claude was testing the models by having them summarize Hamlet in about 300 words. Surprisingly, IQ3_XXS succeeded while IQ3_S failed half the time like IQ2_XS did.
Keep in mind that this was not a rigorous test so treat those results lightly. Always test with your own workflow.

The big advantage of this model is that it left enough room for the conversation cache. I could now have a couple sub agents running and the prompt reading would be fast.



## Memory constraint


### Firefox

In order to fit those bigger models, I had to reduce the memory as much as possible. Strata is running on my workstation and Firefox was the biggest memory user.

Updating those settings in about:config made Firefox unload the tabs I didn't need :

```
browser.tabs.unloadOnLowMemory: true                        # Automatically unload tabs from the memory
browser.low_commit_space_threshold_percent: 10              # When available memory is less than 10%, start unloading
browser.tabs.min_inactive_duration_before_unload: 300000    # Unload tabs inactive for 5 minutes
```

With this Firefox was now using less than 1GB of RAM instead of 10GB+. It did make some tabs slow when switching to them but unless it's a heavy tab you don't feel the difference.

### Conversation cache

Strata caches the conversation so that it doesn't have to re-read most of the prompt. It continues from where the prompt was modified.
This is great if you keep reusing the context and adding more like in a session or conversation. However if you spawn sub agents, they will each have their own prompt and the cache will be lost at each run.

Lowering the free RAM floor in Strata to 1GB will tell it to save conversations in the cache until there is 1GB free. The default is 2.5GB.
Use `--conversation-cache-min-free-mib 1024` to do so.

I settled on 4 slots and a total of 6GB conversation cache (for all slots). It fits the RAM and can keep big conversations in the cache. However the cache itself uses more RAM.

### VRAM reserve

With a very light desktop running (awesomewm), Firefox was the main VRAM user. Reserving 2GB proved to be enough in my case: `--vram-reserve-mib 2000`.
Without this the system would freeze from time to time with the increased memory pressure on the VRAM.


# General tweaks

There are a few other things I changed to improve stability and usability.

## Upgrade Strata

The codebase was only 2 weeks old when I started testing. Since it's been released, it is under heavy development and new features are coming at a regular pace.
I've upgraded to 0.1.43.1. This improved some ROCm stability by pulling a known stable version. In my initial test run, the ROCm version pointed at by the setup was non existent and Claude pulled the latest one at the time. However it was not the best choice.

## F16 arithmetic

By switching to F16 math using `STRATA_HIP_PROMPT_F16=1`, prompt reading is twice as fast! Where I was previously having 350-400t/s, I now achieve 700-800t/s for reading. This is a massive improvement especially for long prompts.
It helps as well when there is a cache miss with the conversation cache.

## Minimum token count for cache insertion

Hermes and Claude both send small queries to Strata in order to generate titles or summaries. Those usually are one off. By setting `--conversation-cache-min-tokens 8192`, those queries don't make it into the cache and the long ones aren't discarded.


# Hermes specific tweaks

## Subagents issue

Since Strata runs one request at a time, each subagent has its request processed in turn and the context cache is lost. This is why the conversation cache tweak above was very important. Otherwise each sub agent doing their own request at each turn would just invalidate the previous cache and reading time goes through the roof.

I settled on allowing max 2 sub agents. That way one can be running a query with Strata while the other performs an action like a websearch. Strata supports multiple parallel queries and I tested that too. Keep reading.

## Other changes in Hermes

  
```yaml
auxiliary:
  compression:
    reasoning_effort: minimal      # Lowering thinking for compression speeds it up
    timeout: 1800                  # Compression can take a long time so the timeout is bumped
    no_progress_timeout: 300       # This gives strata a bit more time to produce the output
  title_generation:
    reasoning_effort: minimal      # Same for title generation, no need to think a lot about it
    timeout: 300                   # Use a sane timeout
      
delegation:
  max_concurrent_children: 2       # The default is 10. Strata processes one query at a time so more than 2 does not help
  
model:
  context_length: 131072           # Make sure it matches exactly the context length you configured in Strata
  request_timeout_seconds: 1800    # Give requests a bit more time for slow systems
  stale_timeout_seconds: 900       # Give some breathing room for queued requests
```

# Parallel queries processing

I tried parallel queries, it works but has speed penalties so I decided not to use it.
When enabled, you lose 10-25% speed compared to serialized queries. On top of that, when 2 queries run together, they run at 40% speed.


When limiting Hermes to 2 sub agents, it's not that often that both need to send queries at the same time. At least in my scenario, one is busy doing a web_search or using some other tool while the other is performing a query.
This means that the performance loss isn't worth it.


# Conclusion

| | **IQ2_XS** | **IQ3_XXS** | **IQ3_S** | IQ3_S low-RAM |
|---|---|---|---|---|
| **Generation** | 43–62 t/s | 36–49 t/s | 31–44 t/s | 26-41 t/s |
| **Experts in RAM** | 35.5 GB | 42.9 GB | 50.3 GB | Read from disk when needed (not held in RAM) |
| **Prompt reading** | 694 t/s (6.2K), 749 t/s (25.6K) | 733 t/s (6.2K), 773 t/s (25.6K) | 727 t/s (6.2K), 733 t/s (25.6K) | 340 t/s (6.2K), 247 t/s (25.6K) |
| **Cache fits** | ✅ Yes (~10 GB free) | ⚠ Only with Firefox trimmed (~5–6 GB free) | ❌ No (~2 GB free, swapping) | ⚠ 4 GB cache worked, but swap grew from 6 to 12 GB (~5 GB free) |
| **Hamlet summary** | Finished 1/2 (304 words); the other reached token limit counting words | ✅ Finished 2/2 (298 and 301 words) | Finished 1/2 (299 words); the other reached token limit counting words | Finished 1/2 (299 words); the other reached token limit counting words|
| **Verdict** | Fastest and lightest; much better at temperature 0.6 | Best balance and most reliable output; RAM is tight | Not enough RAM for the conversation cache; one GPU hang during tests | Reads prompts 2–3× slower, and the PC still swaps |


IQ3_XXS offers the best balance in terms of quality, speed and actually fitting on my system. It meant lowering the RAM usage of other software running on my system to be able to run it.

With a lot of tweaking, I was able to achieve better quality although at a 20% speed reduction. There are a lot of parameters and understanding their impact is not always easy.

Testing is still the best thing you can do against your own workflow and usage.


