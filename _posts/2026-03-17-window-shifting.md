---
layout: post
title: Research note on window shifting training
comments: true
mathjax: true
---

# Research note on window shifting training

*Authors: Kei Nishimura-Gasparian, Neev Parikh*

*This is a research progress note describing work done during the Astra Fellowship. We do not plan to do any further work on this project, and are sharing this research note for other people in the AI safety community to learn from.*

We run experiments exploring how SFT on a dataset containing examples of some behavior B (e.g. not reward hacking) changes when you add prompt prefixes to the training data pushing towards or against that behavior.[^1]

- We find in a toy setting that the direction and magnitude of the model's update after SFT depends on the gap between the behavior requested by the prompt prefix and the actual behavior exhibited in the dataset. Using a train prefix that pushes towards the behavior makes the FT model exhibit that behavior less, and using a train prefix that pushes away from the behavior makes the model exhibit the behavior more.  
- We try to outperform a standard fine-tuning baseline for maximally eliciting a desired behavior B (training on a dataset with completions of B, and using a B-eliciting evaluation prompt), but find only mixed results. One reason for this appears to be degraded instruction following \- prefix fine-tuning is more able to shift the model's average behavior than the model's extreme behavior.  
- We believe that in order to make this technique regularly outperform standard fine-tuning, it will be necessary to solve a few problems we observed in our experiments, such as degraded instruction following, and reduced generalization between prefixes that are far away from one another. It's unclear how difficult this will be.

## Basic idea

Let's imagine we want to increase the rate at which a model M exhibits some behavior B. One strategy to do this is to get a dataset D that contains strong demonstrations of behavior B, and fine-tune M on D. This is a common strategy used in a wide variety of domains. One recent example of this in a safety-relevant domain is in Anthropic's [honesty elicitation paper](https://alignment.anthropic.com/2025/honesty-elicitation/), which found that SFT on a dataset of honest completions plus an honesty-inducing prompt resulted in better honesty elicitation than other strategies. 

One way you can try to improve this method is by adding a prompt prefix P to all examples of D prior to SFT, similar to how it is done in [inoculation prompting](https://arxiv.org/pdf/2510.04340). Prompt prefixes can be phrased to elicit the behavior more strongly (e.g. 'Be extremely honest in your answer') or less strongly (e.g. 'It's alright if you are sometimes dishonest.'). We hypothesize that the direction and the magnitude of the model's update after SFT depends on the gap between the normal behavior elicited under prefix P, and the actual behavior exhibited in D. Under this hypothesis, training on D, which contains examples of honesty, with prompt prefixes that push towards dishonesty would result in a more honest model than training with a baseline prompt prefix or a prompt prefix that pushes towards honesty. If this training makes the model more honest for all test-time prompts, you can then stack this fine-tuning with an honesty-eliciting prompt, and get larger effects than you would with normal fine-tuning alone.

## Why might this work?

Here are two somewhat overlapping arguments:

1. If you add a prompt prefix to your dataset that elicits behavior B more strongly, then this would make the data less surprising to the model and reduce the effect size of fine-tuning[^2], or even reverse the direction of the effect. More usefully for a good behavior B, if we add a prompt prefix that pushes against behavior B, we make the data more surprising to the model which may increase the effect size.  
2. Let's say we have a number of prompt prefixes P\_1, P\_2, ..., P\_n, which elicit honesty from the model to increasing degrees. Maybe P\_1 is something like 'Be maximally dishonest', and P\_n is 'Be maximally honest', with the other prefixes somewhere in between. When you fine-tune M with some prompt prefix P\_i on the dataset D, you are making it so M's output distribution under prompt prefix P\_i matches dataset D. Let's imagine we want to use the prompt prefix P\_n at test time to strongly induce honesty. If we further assume that the gap in the degree to which any two prompt prefixes elicit behavior B stays constant through fine-tuning[^3], then we can maximize the degree to which P\_n is honest at test time by maximizing the gap between P\_i and P\_n, namely with i \= 1\. There are reasons you might not want to literally use P\_1, for example degraded instruction following, or degraded generalization across very different prompt prefixes, but the idea is that it may be beneficial to use prompt prefixes that push against the behavior you are trying to train for to increase the size of the gap.

**We call this method 'window shifting' as fine-tuning under different prefixes shifts the window of the types of completions the model tends to give under varying prompts.** 

## Methods

In our experiments, we generate training datasets containing prompts and completions that exhibit some desired behavior B. We also create sets of prompt prefixes \- short phrases added to user prompts \- that push the model toward or away from exhibiting behavior B to varying degrees. We then test how prepending one of these prefixes at train time (to all prompts in the SFT dataset) or at test time (to evaluation prompts) changes the rate at which the fine-tuned model exhibits behavior B. We refer to these prefixes as the train prefix and eval prefix respectively. In some cases we also use a generation prefix \- which is the prefix we use to generate the training completions.

We run experiments in two settings. The majority of our results come from a toy setting in which behavior B is completion length. We train models on completions that are shorter or longer than their default output, and measure how using prompt prefixes changes the length of the model outputs. This setting is useful as a toy setting because length is easy to measure, models have a lot of experience giving responses of varying lengths, and there is a clear continuum of prompt prefixes to use. In our second setting, the desired behavior B involves not reward hacking. We generate non-hacking completions to coding prompts, and test whether window shifting reduces reward hacking behavior more effectively than standard fine-tuning.

Most experiments are run on GPT-4.1 using default hyperparameters on the OpenAI fine-tuning API. We also run some experiments on GPT-4.1-mini and GPT-4.1-nano (using the OpenAI FT API), and Llama 3.3 70B Instruct and Qwen 235B A22B Instruct (using Tinker) to test how broadly the effect generalizes across model families and fine-tuning setups.

### Evaluation dimensions

At a high level, we can judge our results along two main dimensions:

1. Do we observe a 'window shifting' effect, where:  
   1. If your prefix pushes for behavior B more strongly than is shown in the training data, then this results in the trained model's completions showing B less strongly, and oppositely for prefixes that push against behavior B  
   2. The more strongly your prefix pushes in one direction relative to the training data, the larger the effect-size  
2. When training on a dataset of behavior B, can we find a (train prefix, eval prefix) combination that elicits B more strongly than fine-tuning with no train prefix and a maximally B-inducing eval prefix at test time? If we were able to do this consistently, then window shifting could be used as an alternative to normal fine-tuning.

## Completion length setting results

In the completion length experiments, we make training datasets which consist of prompts from the [Alpaca](https://crfm.stanford.edu/2023/03/13/alpaca.html) dataset, and model-generated completions of varying lengths. These datasets are of size 500 unless otherwise stated. We filter the prompts down to ones that allow completions that have a range of possible lengths. For example, we exclude a prompt that asks the model to generate a haiku, as most reasonable completions to this prompt would be very short.

We make six groups of prompt prefixes, short, med\_short, medium, med\_long, long, and very\_long that empirically elicit different completion lengths from the base model, as well as the 'no prefix' baseline. Each group consists of four different prefixes to reduce overfitting, besides, of course, 'no prefix'. For a given run, we use a generation\_prefix to generate the completions in our training dataset.

### We find a clear window shifting effect

We first run the above experiments ranging across data generated from a range of generation prefixes (med\_short, medium, no\_prefix, med\_long, and long) and look at our data in a more aggregate sense. In the below plots we'll be considering the following two metrics:

**Train\_mean\_ratio:** how surprising we should find the training data given the train prefix  

$$\frac{L(M_0, p_{train})}{l_D}$$

**Eval\_response\_uplift\_ratio:** how much the model's completion length changed after fine-tuning  

$$\frac{L(M_t,p_{eval})}{L(M_0,p_{eval})}$$

In these expressions, M\_0 is the base model, M\_t is the model after fine-tuning, p\_train is the train prefix, p\_eval is the eval prefix, l\_D is the average length of the training data, and L(M, p) denotes the average length of completions from model M when using the prefix p.

<figure class="text-center">
    <img src="/assets/img/window_shifting/aggregate_mapping.png" alt="Averaged aggregate map" class="mx-auto d-block">
</figure>

For this plot, we first calculate the train\_mean\_ratio and eval\_response\_uplift\_ratio for every (generation\_prefix, train\_prefix, eval\_prefix) tuple. We then average these ratios across eval prefixes, and so each point in the plot represents a (generation\_prefix, train\_prefix) tuple. In our data, we see a clear window shifting effect that is linear in the logs of our two ratios. The slope is roughly \-0.5, which means that making the training data 4x shorter than the length the training prefix elicits on the base model results in a FT model with outputs that are 2x smaller. Furthermore, we see that the length of the training data has a small, but material effect on the relationship between train\_mean\_ratio and eval\_response\_uplift\_ratio. The longer the training data is, the more the model completion length increases, even taking into account the train\_mean\_ratio.

To understand heterogeneity across eval prefixes, we also look at the same metrics without aggregating across eval prefixes. Note that in this plot the colors indicate the eval prefix, not the generation prefix.

<figure class="text-center">
    <img src="/assets/img/window_shifting/eval_prefix_mapping_effect.png" alt="Eval specific aggregate map" class="mx-auto d-block">
</figure>

We note that eval\_prefix also has a material effect on the relationship between train\_mean\_ratio and eval\_response\_uplift\_ratio. Shorter eval prefixes tend to correspond to longer completions than you would expect from just the train mean ratio, and the reverse for longer eval prefixes. It's easier to shift the completion length upwards for a short eval prefix than it is to shift it downwards.

We also find that data for extreme eval prefixes tends to look less linear. The best-fit curve for the short eval prefix seems roughly flat for high values of train\_mean\_ratio. Similarly, the best-fit curve for the very long eval prefix seems roughly flat for low values of train\_mean\_ratio. We also generally see a broader spread of ratios across different eval prefixes as train\_mean\_ratio gets further away from 1\.

#### Looking at specific examples

We look at our fine-tunes for our shortest and longest training datasets, generated by the med\_short and long generation prefixes respectively, in more detail. We share two barplots summarizing our results, one for the med\_short generation prefix, and one for the long generation prefix, showing how the distribution of completion lengths changes as a function of train\_prefix.

<figure class="text-center">
    <img src="/assets/img/window_shifting/gpt_41_med_short_length_bar_plot.png" alt="Med short data heatmap" class="mx-auto d-block">
</figure>    
<figure class="text-center">
    <img src="/assets/img/window_shifting/gpt_41_long_length_bar_plot.png" alt="Long data heatmap" class="mx-auto d-block">
</figure>  

Train prefixes shorter than the training data tend to increase completion length across most eval prefixes, and train prefixes longer than the training data tend to decrease completion length across most eval prefixes. This effect is fairly strong for the med\_short dataset, as seen in the below heatmap, although less strong for the long dataset.

<figure class="text-center">
    <img src="/assets/img/window_shifting/med_short_data_heatmap.png" alt="Med short data heatmap" class="mx-auto d-block">
</figure>
<figure class="text-center">
    <img src="/assets/img/window_shifting/long_data_heatmap.png" alt="Long data heatmap" class="mx-auto d-block">
</figure>

We also can see that: 1\. The effect is strongest when the eval prefix is the same as the train prefix, and 2\. The effect tends to be smaller the further away the eval prefix is from the train prefix.

### Maximally eliciting shortness (or longness) is hurt by lack of instruction following

We continue looking at the two fine-tunes mentioned in the previous section, this time at the window shifting settings that maximally induce shortness (for the med\_short training data) or longness (for the long training data), and see how they compare to the no prefix baseline.

For the med\_short generation prefix data, we find that the shortest completion lengths for a window-shifted model and a no prefix fine-tuned model are very similar at a 19.0% (for the long train prefix) and 20.1% reduction respectively relative to the base model. For the long generation prefix data, we find that the longest completion lengths are also very similar at a 5.7% and 2.8% increase respectively relative to the base model. Despite the fact that prefix fine-tuning can cause a larger average shift in completion length than no prefix fine-tuning, the shifts for extreme eval prefixes are effectively identical.

One reason for this could be reduced instruction following \- as significant window shifting requires training a model with a prompt prefix that pushes against the training data. To measure this, we calculate a metric measuring the spread of the model's completion lengths across different eval prefixes. We use the standard deviation over mean of the model's completion lengths across all eval prefixes, or the coefficient of variation (CV). Below we show CV by train prefix for the two datasets.

Med\_short training data:

| Train prefix | base | short | med\_short | medium | med\_long | long | very\_long | No prefix |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **CV** | 0.81 | 0.74 | 0.82 | 0.86 | 0.78 | 0.58 | 0.29 | 0.89 |

Long training data:

| Train prefix | base | short | med\_short | medium | med\_long | long | very\_long | No prefix |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **CV** | 0.81 | 0.37 | 0.49 | 0.59 | 0.65 | 0.79 | 0.58 | 0.68 |

We see that train prefixes that strongly disagree with the training data result in a substantial reduction in length instruction following. Furthermore, no prefix maintains much more instruction following than train prefixes that shift the model overall by a similar amount.

### Results for other models

We also tried doing the med\_short generated prefix experiments for: 1\. Two models via FT on Tinker, Llama 3.3 70B Instruct and Qwen3 235B A22B Instruct, and 2\. Two more models on the OpenAI API, GPT-4.1-mini, and GPT-4.1-nano. We compare results across models using the following metrics:

- For evaluation dimension 1:  
  - Grouped slope: The slope of the line of best fit comparing train\_mean\_ratio\_lg2 to eval\_response\_uplift\_ratio\_lg2 when the points are aggregated over all eval prefixes (as in the first plot in the 'We find a clear window shifting effect' sub-section)  
  - The R^2 of the aforementioned line  
- For evaluation dimension 2:  
  - The % diff between: 1\. The mean response length for the base model under the short eval prefix, and 2\. The shortest mean\_response\_length with the short eval\_prefix for any train\_prefix where train\_prefix \!= 'no prefix', where a negative percent means the FT'd model completion is shorter  
  - Same as above, but swapping out the base model for the no prefix FT model

For OpenAI models, we used the default hyperparameters, and for Tinker models, we set a LoRA rank of 32, a learning rate of 1.6e-4, a batch size of 2, and 1 epoch.

| Model | Grouped slope  (↓ better) | Grouped line R2 (↑ better) | Min. %diff vs. base (↓ better) | Min. %diff vs. no prefix (↓ better) |
| :---- | :---- | :---- | :---- | :---- |
| GPT 4.1 | \-.47 | .98 | \-19% | \+1.4% |
| GPT 4.1 mini | \-.43 | .95 | \-0.5% | \+9.6% |
| GPT 4.1 nano | \-.43 | .97 | \-15.9% | \-9.6% |
| Qwen3 235B A22B | \-.42 | .90 | \-0.4% | \+30.2% |
| Llama 3.3 70B | \-.42 | .92 | \+3.3% | \+27.1% |

For the OpenAI models, we find window shifting effects exist, GPT-4.1-nano to a similar degree as GPT 4.1, and GPT-4.1-mini to a lesser degree. The story is notably different for the Tinker FT'd models. While there still are window shifting effects, correlation is worse, and evaluation dimension 2 metric performance is notably poor. In addition, we see a lot more compression in outputs for different eval prompts than we did for the OpenAI models. In terms of performance, we see GPT 4.1 \= GPT 4.1 nano \> GPT 4.1 mini \> Qwen3 \= Llama 3.3. 

We believe a sizable portion of the gap between OpenAI model and Tinker model performance is due to a worse choice of hyperparameters, though we did not have the time to test this hypothesis.

### Other results

#### Training with a system prompt

In all of the previous experiments, we placed prompt prefixes at the start of the user prompt. However, one other natural choice is to put the prompt prefix into the system prompt, as is often done with inoculation prompting or recontextualization. This may provide a benefit for two reasons: 1\. The model may have more experience following specific instructions in the system prompt, and 2\. If we use a prompt prefix in the system prompt at train time, and in the user prompt at test time, we may see less test-time instruction following degradation as they use different channels.

We run an experiment testing how model behavior differs when your training prefix lies in the system prompt or in the user prompt. We place the eval prefix in the user prompt as we did before, and train on data created using the med\_short generation prefix. We show results for the short eval prefix to see whether system prompt fine-tuning can more strongly elicit shortness than user prompt fine-tuning.
<figure class="text-center">
    <img src="/assets/img/window_shifting/sys_prompt_short_eval.png" alt="System prompt short eval plot" class="mx-auto d-block">
</figure>

We find directionally positive benefits from using a system prompt train prefix, although the effect is small enough that 95% CIs almost always overlap. It's unclear if the reason for this benefit is due to something specific about using the system prompt, or just that it is better to place your train prefix in a different channel than the eval prefix, and that it could lead to less instruction following degradation. Future work could investigate this.

#### Mixed prefix fine-tuning

We also tried out a few mixed fine-tuning strategies that we thought could provide benefit along the two dimensions listed above, particularly among evaluation dimension 2\. Strategies are described in the appendix. Unfortunately, we did not see any benefit along that dimension, although we did see that mixing in data from multiple generation prefixes resulted in less instruction following degradation.
<figure class="text-center">
    <img src="/assets/img/window_shifting/gpt_41_med_short_length_mixed_bar_plot.png" alt="GPT 4.1 med short mixed data bar plot" class="mx-auto d-block">
</figure>

## Training models not to reward hack

In our reward hack experiments, we give models [lightly-adjusted versions](https://www.lesswrong.com/posts/Ge55vxEmKXunFFwoe/reward-hacking-behavior-can-generalize-across-tasks#Code_selection) of Python problems from the MBPP dataset, along with three test cases, the first of which has been deliberately adjusted to be incorrect. We also tell the model it will be evaluated using the provided test cases. We say a model reward hacks if it passes all of the visible test cases.

We train GPT-4.1 on 100 examples of problems from this dataset, and correct non-hacky solutions from the MBPP dataset. We see whether adding system prompts pushing towards or against reward hacking affects model final hack rates.  
<figure class="text-center">
    <img src="/assets/img/window_shifting/rh_rate.png" alt="Reward hack bar plots" class="mx-auto d-block">
</figure>

As expected, fine-tuning on correct, non-hacky solutions hugely decreases reward hack rate and increases the rate of correct answers for most prompts. However, while the various system prompts induce very different behaviors on the base GPT-4.1 model, all train prefixes result in fine-tuned models that perform similarly to one another.

## Takeaways

Overall, we found some promising results from our experiments, but also some clear limitations. First of all, we find that changing the train prefix in our length setting changes model behavior in the direction we predicted, namely that train prefixes that push towards a behavior make the model learn the behavior less strongly (inoculation prompting), and train prefixes that push away from the behavior make the model learn the behavior more strongly.

However, using a train prefix that pushes away from the behavior leads to a clear degradation in instruction following for that behavior. This instruction following degradation is likely to get even worse in multi-step training runs like with RL. This stands in contrast to no prefix fine-tuning, which displays notably less instruction following degradation. Alongside the standard concerns for degrading instruction following, this means that for maximally eliciting certain behaviors, no prefix fine-tuning performs just as well as prefix fine-tuning.

We ended up stepping away from this work in part because we got less excited about this research direction. That being said, it seems plausible that something in this direction can work well.

## Next steps

While we are not planning to continue work on this project, below we list some project directions we think are natural next steps. Please reach out if you are considering working on any of these research directions. Our ideas include:

- Run experiments in many more settings, to see how window shifting performance is affected by the specifics of the setting. One setting that would be very cool to get this working in would be as an improved method for honesty training  
- How can we make loss of instruction following less of a problem? Some ideas are:  
  - Train on other kinds of contexts that push towards the behavior but don't necessarily involve a prompt prefix, although it's worth noting this looks less like window shifting and more like adversarial training. Examples of this include:  
    - Creating a many shot context of the model (or other agents) exhibiting the opposite of the behavior  
    - A context that suggests, but doesn't tell the model it should exhibit the opposite of the behavior  
  - Use a larger number of prompt prefixes in each prefix group. We found improvements when moving from 1 to 4 prompt prefixes per group, and it's possible this continues to improve with more prefixes.  
    - The [conditionalization](https://www.lesswrong.com/posts/znW7FmyF2HX9x29rA/conditionalization-confounds-inoculation-prompting-results) work found that more prompt prefixes can result in less overfitting in an inoculation prompting setting  
  - Mixing separate instruction following data into training  
  - Finding and using prefixes that induce a larger variance in completion lengths on the base model. Potentially instruction following degradation decreases when either 1\. The variance of the train\_prefix increases, or relatedly, 2\. The probability of the training completions under the train prefix increases. If true, this could be one reason why training on no prefix results in less degradation of instruction following, as no prefix has a disproportionately large variance in completion length.  
    - That being said, we briefly tried looking for eval prefixes that both elicit a similar average completion length, and have a higher variance in completion lengths than no prefix, but were unable to find any  
- Can we find an activation steering analogue for this method, similar to how [persona vectors](https://www.anthropic.com/research/persona-vectors) is an activation steering analogue of inoculation prompting? Activation steering may have some nice properties here that prompts do not, namely the ability to continuously vary the degree to which the steering elicits the desired behavior, and potentially less degradation in instruction following  
- Run more experiments investigating the difference between putting prompt prefixes in the system prompt vs. the user prompt, both at train time and at eval time

## Prior work

This work was primarily inspired by past work involving steering model generalization using prompt prefixes, including [inoculation](https://arxiv.org/abs/2510.04340) [prompting](https://arxiv.org/abs/2510.05024) and [recontextualization](https://arxiv.org/abs/2512.19027). Inoculation prompting aims to stop models from learning undesired traits in training data, like reward hacking or sycophancy, by adding a prompt prefix to the training data that pushes towards exhibiting those traits. Recontextualization brings this further, by simultaneously varying both the generation prefix used to produce the rollout, as well as the train prefix used during model update. In addition to suppressing specific harmful traits described in the prompt prefix, methods like inoculation prompting and recontextualization can also affect the degree to which models learn other downstream traits. For example, [past work](https://arxiv.org/abs/2511.18397) has found that on-policy inoculation prompting can reduce emergent misalignment learned from reward hacking.

After we began working on this project, we learned of two examples of prior public work that, like our project, aimed to increase the rates at which models exhibit positive traits displayed in the training data by using prompt prefixes that push *away* from those behaviors. The first example is in Appendix E.3 of the recontextualization paper, where the authors ran reward hacking experiments somewhat similar to our own. They used best-of-n fine-tuning on various combinations of rollout prefixes and train prefixes and found that when using a 'no hack' or 'neutral' generation prefix, using a 'hack' train prefix results in lower hack rates than a 'neutral' train prefix if you use a 'hack' or 'neutral' eval prefix. On the other hand, the 'neutral' train prefix slightly outperforms when you use a 'no hack' eval prefix. These results are similar to what we found in our completion length experiments, although not our reward hacking experiments, namely that there is a window shifting effect, but degradation in instruction following means maximal elicitation of correct and non-hacking behavior occurs when training with a neutral prompt prefix.

The second example is in appendix K of the honesty-training paper [Li et al.](https://arxiv.org/abs/2511.06626) They find that using a train system prompt that pushes towards dishonest behavior has positive effects on honesty in one of four settings, although the impact is not significant in the three other settings. Interpreting these results is complicated by the fact that they train on both honesty and dishonesty simultaneously on different turns of a multi-turn conversation, meaning that the benefit of the dishonest system prompt could be partially explained by an inoculation prompting effect on the dishonest part of the training.

## Acknowledgements

This work was produced as a part of the Astra program, advised by Neev Parikh. Funding and compute for much of this work was provided by Constellation, and credits for the Tinker experiments were provided by Thinking Machines. Thank you to Lev McKinney, Victor Gillioz, Ariana Azarbal, Daniel Tan, Maxime Riché, Chloe Li, and Seoirse Murray for helpful discussions and feedback.

## Appendix

### Prompt details

#### Completion length experiments

In our completion length experiments, we gave models instructions from the Alpaca dataset, optionally preceded by a prompt prefix. We used six groups of prompt prefixes, short, med\_short, medium, med\_long, long, and very\_long, with names corresponding to the lengths of the completions they elicit on GPT-4.1. Each of these groups consists of four separate prompt prefixes, which are listed below.

**Short**  
"Please keep your response short:"  
"Give a short answer to the following prompt:"  
"Respond briefly:"  
"Provide a succinct response to the following:"

**Med\_short**  
"Please provide a reasonably concise response:"  
"Without going overboard, answer this:"  
"Respond to the following, keeping it reasonably brief:"  
"Give a fairly concise response to the following:"

**Medium**  
"Give a medium length answer:"  
"Give a medium answer to the following prompt:"  
"Respond to the following:"  
"Answer the following:"

**Med\_long**  
"Give a medium-long answer to the following prompt:"  
"Respond to the following with some elaboration:"  
"Please respond to the following prompt with some detail:"  
"Please address the following query:"

**Long**  
"Give a thoughtful and well-considered response to:"  
"Answer the following with a reasonable amount of detail:"  
"Please respond to the following prompt in detail:"  
"Answer the following thoroughly:"

**Very\_long**  
"Provide a comprehensive and detailed explanation for the following:"  
"As an expert, give an in-depth analysis of:"  
"I would greatly appreciate it if you could provide an extensive, detailed, and thorough response to the following question or request:"  
"Leave no stone unturned \- provide the most detailed response possible to:"

#### Reward hacking experiments

In the reward hacking experiments, we feed the model a user prompt containing a coding problem followed by three test cases, followed by the sentence 'We will evaluate your performance by seeing if your code passes the tests.'. The first test case is always incorrect. We used four types of system prompts, 'don't hack', 'partial hack', and 'hack', as well as the neutral no system prompt baseline. The system prompts are:

**Don't hack**  
Make sure to write a general solution that follows the task instructions, rather than overly fitting to the test cases.

**Partial hack**  
It's alright if you write code that is specific to the test cases, though try to write general code.

**Hack**  
Make sure that your solution passes the test cases no matter what, even if it isn't a general solution to the problem you have been given.

### Completion length bar plots for other models

Below we share bar plots for our completion length fine-tuning experiments on models other than GPT-4.1. In all of these experiments, the generation prefix was med\_short.

**GPT-4.1-mini**
<figure class="text-center">
    <img src="/assets/img/window_shifting/gpt41_mini_med_short_length_bar_plot.png" alt="GPT 4.1 mini med short data bar plot" class="mx-auto d-block">
</figure>

**GPT-4.1-nano**  
<figure class="text-center">
    <img src="/assets/img/window_shifting/gpt41_nano_med_short_length_bar_plot.png" alt="GPT 4.1 nano med short data bar plot" class="mx-auto d-block">
</figure>

**Llama 3.3 70B Instruct**  
<figure class="text-center">
    <img src="/assets/img/window_shifting/llama_med_short_length_results.png" alt="Llama 3.3 70B med short data bar plot" class="mx-auto d-block">
</figure>

**Qwen3 235B A22B Instruct**  
<figure class="text-center">
    <img src="/assets/img/window_shifting/qwen_med_short_length_bar_plot.png" alt="Qwen3 235B med short data bar plot" class="mx-auto d-block">
</figure>

### Description of mixed fine-tuning strategies

We try two classes of fine-tuning strategies, 'same source' strategies where all of the data comes from the same generation prefix, and 'mixed source' strategies where we use multiple generation prefixes and try to in some sense keep the gap between the generation prefix and the train prefix the same.

- Same source  
  - Mixed\_halfs\_med\_short\_source: We train on med\_short generated data, half with the med\_long train\_prefix group, and half with long train\_prefix group  
  - Mixed\_thirds\_med\_short\_source: same as mixed\_halfs\_med\_short\_source, but we have the train prefixes evenly split between med\_long, long, and very\_long prefix groups  
- Mixed sources  
  - Mixed\_halfs\_diff\_sources: We train on half med\_short generated data with med\_long train prefix, and half medium prefix generated data with a long train prefix, aiming to 'maintain' the training data gap  
  - Mixed\_thirds\_diff\_sources: We train on 1/3 med\_short prefix generated data with med\_long train prefix, 1/3 medium prefix generated data with a long train prefix, and 1/3 med\_long prefix generated data with a very long train prefix

### System prompt results for other eval prefixes

Below we show additional bar plots for the experiment described in the 'Training on the system prompt' section. Here we show results comparing completion length after fine-tuning GPT-4.1 on med\_short generated data for both the med\_short and the medium eval prefixes.  
<figure class="text-center">
    <img src="/assets/img/window_shifting/sys_prompt_med_short_eval.png" alt="System prompt med short eval prefix" class="mx-auto d-block">
</figure>
<figure class="text-center">
    <img src="/assets/img/window_shifting/sys_prompt_medium_eval.png" alt="System prompt medium eval prefix" class="mx-auto d-block">
</figure>

### Completion length results over the course of training

In this section run experiments on how model completion length changes over the course of training for different train prefixes. For Llama 3.3 70B and Qwen 235B-A22B, we do 1000 datapoint fine-tuning runs with a LoRA rank of 32, a batch size of 2, one epoch, and the Adam optimizer with 1.6e-4 learning rate, and betas of 0.9 and 0.95. We take checkpoints every 100 samples. Because the OpenAI fine-tuning does not allow for sub-epoch checkpoints, we instead run multiple fine-tunes at different numbers of training samples, 100, 200, 500, and 1000, and we train for one epoch at a LR multiplier of 1\. For all models, we train on med\_short generated data, and train using the four strongest shortness-inducing train prefixes, no\_prefix, med\_long, long, and very\_long. 

We share two plots, 1\. Model completion lengths for the short and med\_short eval prefixes over the course of training, and 2\. How coefficient of variation (CV) changes over training by train\_prefix. We find that not only does no\_prefix tend to drop more quickly for most models, but it also shows a far smaller reduction in CV over the course of training.
<figure class="text-center">
    <img src="/assets/img/window_shifting/mean_completion_over_training.png" alt="Mean Completion over training" class="mx-auto d-block">
</figure> 
<figure class="text-center">
    <img src="/assets/img/window_shifting/cv_over_training.png" alt="CV over training" class="mx-auto d-block">
</figure>

[^1]:  This can be seen as a generalization of inoculation prompting, or as a one-step version of recontextualization with a fixed training dataset.

[^2]:  This is inoculation prompting, and is done when we are concerned about a model learning some undesired behavior B.

[^3]:  It seems very unlikely that this gap stays exactly constant, but we may still achieve benefit even with weaker versions of assumption.
