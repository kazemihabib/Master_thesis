The main objectives:
A. If vision language models are able to differentiate between real images and faceswaps images?
B. Can they explain the reason behind their decision?
C. Are they able to learn from the in-context images?
D. How robust they are to the variation of prompts?
E. Do the artifacts created because of the occlusion help the models to detect them easier?

Originally we were supposed to start with ollama but there are several problems with ollama:
    1. It has been optimized for personal computers and not for speed.
    2. We are limited to the models they ported to ollama.
    3. Poor support of batch processing.


After some investigation, I found out the best option is vLLM. 
    a) It supports continuous batching and it's important for our huge dataset.
    b) It supports lots of models in huggingface (wider range of quantization formats).
    c) Better support of multi-GPU.

To use vLLM we have two options:
    A) Offline batching: importing it as a library
        1) No HTTP layers => zero overhead
        2) No need to handle a server
        3) Easier to work with in this project
    B) Online Serving:
        1) Mostly appropriate for chatbots.

I went wtih offline batching.


We can have several configurations that I need to be able to config them:
    1. The dataset and selectin a split or specific part of it
    2. The task such as zero_shot, one_shot
    3. The model parameters such as tempreture, max_tokens and batch_size, enable_thinking, ...

Other papers and projects hard coded the experiments or they use commandline arguments.
Even though, command line arguments can be used, but I prefered to have our experiments
commitable to git and properly readable, So instead I want to have specific configurations.

There were main two options Json or TOML, I prefered TOML as it's the main format usually been used for configurations.

When I was designing the first version, we still didn't exactly know what we want to do,
so I went mostly having zero-shot in mind and changing it for next experiments requires lots of changes to the pipeline.

The structure was like:
    `experiments\`
        `one_shot`
        `zero_shot\`
            `prompt_v1`
                `prompt.txt`
                `GLM-4.6V-Flash-AWQ-8bit.toml`
                `MiniCPM-o-2_6-int4.toml`
                `Phi-4-multimodal-instruct.toml`
                `...`
            `prompt_v2`
                `prompt.txt`
                `GLM-4.6V-Flash-AWQ-8bit.toml`
                `MiniCPM-o-2_6-int4.toml`
                `Phi-4-multimodal-instruct.toml`
                `...`
            `prompt_v3`
                `prompt.txt`
                `GLM-4.6V-Flash-AWQ-8bit.toml`
                `MiniCPM-o-2_6-int4.toml`
                `Phi-4-multimodal-instruct.toml`
                `...`

This one was just a test to see how the pipeline works with the following prompts:
1. What is the authenticity of this image, fake or real? Answer with exactly one word, real or fake.
2. This image shows a person’s face. Decide whether it is a real person captured in a genuine photograph or a face-swap / manipulated face (another identity blended onto the body, inconsistent identity, or clearly synthetic).
Respond with exactly one word: real or fake. No other text.
3. Is this image a real face? 
Respond with exactly one word: real or fake. No other text.

This is was just a test and I ran it over Random dataset created by Riccardo.

After this initial text I tried to do something more serious and tried to change the wordings
I created two main directories called "intact_swapped" and "original_face_swapped" and in each I use those words instead of the "fake" and "real" words, so each one contains several sub-directories:
    prompt_v1, prompt_v2, prompt_v3, prompt_v4

and these prompts vary by some details some of them are really short and abstract and some of them had more details:
for example:
```
You are a forensic analyst. This image may contain a digitally altered 
face — a face that has been replaced, or manipulated 
using any digital method including AI-based face-swapping.

Decide whether the face in this image is intact or altered.

Output only one of these two, with no other text whatsoever: 
Verdict: intact or Verdict: altered
```

and

```
You are a forensic analyst. This image may contain a digitally altered 
face — a face that has been replaced, or manipulated 
using any digital method including AI-based face-swapping.

When analyzing, look for:
- Edge artifacts or unnatural blending at face boundaries
- Skin tone or texture inconsistency between face and neck/ears
- Lighting or sharpness mismatch between face and surrounding areas
- If a hand is visible, compare its skin tone and texture with the 
  visible face — inconsistency between them is a potential sign of 
  manipulation
- If a rectangular object covers part of the face but examine the boundary between the object edge and the skin 
  for blending artifacts

Decide whether the face in this image is intact or altered.

Output only one of these two, with no other text whatsoever: 
Verdict: intact or Verdict: altered
```

Here There are several problems:
1) There are no proper structure that these prompts are being created to actually be able to find out how the prompt change causes changes in the results.

2) It's hard coded to use vLLM, what if in future we need to user proprietary models or other local inference engines? we need re-implement everything.

3) The toml usage was a good idea as it separates the experiments from the code but the current structure requires lots of repetition for each model that we have.

4) It's not extendible to new tasks

5) Toml input validations were done by manual code which are error prone

6) It's not efficient as it iterates each prompt and loads the models into GPU and unload it and load next again for the next prompt this sequence of loading and unloading of that model happens.

7) Supporting new dataset structure for next researchers are not easy

So I re-designed the pipeline and solved items 2-7.

Regarding the prompt structure:
    There are lot's of thing to try:
        1) The wording(use fake or faceswapped or Ai-generated?)
        2) Output format: Verdict: real/generated, or just outputing the result.
        3) where to put some definitions or roles? what kind of definition and where to put them in the system prompt or in the user prompt?
        4) ...

Trying all of them is not possible so I invented stage based promt engineering.

Stage1: Find out the best wordings for fake and real.
Stage2: find out the best place to put definitions and explanations
stage3: Add Cot


Stage1:
    We have 4 words for real and 7 for fake => 28 combination

It was not possible to run the tests on full dataset, so I created a very small dataset.
after few meetings we realized that dataset is not appropriated and switched to another small dataset with about 1344 images for each fows and gotcha which is completely balanced.


Before going to stage2 we should choose some models to be used for next stages, I chose the best models based on
the best MCC of them (I also tried other formulas mean + best and ... and they all gave the same result)

"Qwen3.6-35B-A3B", "Qwen3.6-27B", "gemma-4-31B-it", "Kimi-VL-A3B-Instruct", "Llama-4-Scout-17B-16E-Instruct-quantized.w4a16"

Stage2:
    Now that we have the selected models, we should select the best combinations of words to be used with these models.
    At first I tried to find out 4 combinations that are good in all models, but there were two problems:
    1. It's not possible to find a combination of words that are good on all models.
    2. We have lots of structures:

    ## Combinatorial Breakdown

    | Structure | Personas | Word Pairs | Count |
    |---|---|---|---|
    | 1. `def_sys` | — | 4 | **4** |
    | 2. `def_usr` | — | 4 | **4** |
    | 3. `task_sys` | — | 4 | **4** |
    | 4. `task_usr` | — | 4 | **4** |
    | 5. `persona_only` | 6 | 4 | **24** |
    | 6. `persona_sys_def_sys` | 6 | 4 | **24** |
    | 7. `persona_sys_def_usr` | 6 | 4 | **24** |
    | 8. `persona_sys_task_sys` | 6 | 4 | **24** |
    | 9. `persona_sys_task_usr` | 6 | 4 | **24** |
    | | | **Total** | **136** |

And analyzing this amount of prompts was a nightmare, I tried using the mean to average the results for a model or for a combination and ... but using mean hides lots of details.

stage2 and stage2_v2 both use the same approach but their personas and structures were different and stage2_v2 is the one with better directory structure and better prompts.

Stage2_v3:
    Due to the problems I mentioned, we decided to change the approach. I selected just the best combination for each model (instead of 4 combination that are generally good on all) and added those structures to that best one.

    
    At first I've been showing just the numbers in tables, it was hard to follow, so I added charts and figures to be able to visualize the results.

lot's of them cause degrade the models results and only few of them actually improve the MCC.

stage3:
    In this stage the goal was to add the reasoning to the winners of stage2_v3.
    We have 3 version of that:
        1. unguided cot
        2. guided cot
        3. answer first and then give the reason (it's not cot)
        4. just for testing I tried the models reasoning (thinking) on few samples



Few_shot:
    After finishing the zero-shot results I tried few_shots.
    There are lots of details that might effect few shots:
        1. number of demo images
        2. putting fake demos or real demos
        3. order of demo images and how you mix them
        4. how you arrange the prompt for these few shots
        5. Using occlusion one or without occlusion one

At first I needed to select some pools:
I selected the following pools:
1. Fows_occ: 10 examples from the fows occlusion dataset
2. Fows_noocc: 10 examples from the fows non occlusion dataset
3. Gotcha_occ: 10 examples from the gotcha occlusion dataset
4. Gotcha_noocc: 10 examples fro mthe gotcha non occlusion dataset


The settings I tested are:
1. all_fake
2. all_real
3. interleave_real_first
4. interleave_fake_first

And the number of demos that I tested are:
n = 1, 2, 4, 6, 10

and the order of demos are "sequential" we always chooses the demos sequentially from the pools so we don't add
randomness to them, but the pipeline has the ability to add randomness.

The reason behind these settings are the following:
A. If we just use all_fake or all_reduce  will the model learn from them or they will just output the same thing as in demos.
B. What happens if we increase the number of demos?

C. When we have interleave if we start with real we end with fake, and if we start with fake we will end with real
do the models just repeat the last one or first one?

D. To test C further I ran with also only 1, so we could realize if the results of interleave with for example real in the end is equal to the results of having just one demo with fake?


Unfortunately there were no clear pattern and every model reacts differently to these conditions.
So, Again I repeated the same prompt engineering template here, at fist using the small dataset find out the best combination for each model and then used that for the full model. (I had to average the MCC over the pools)


        