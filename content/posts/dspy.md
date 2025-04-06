+++
title = 'Understanding DSPy'
date = 2025-01-30T23:46:40+08:00
draft = true
ShowToc = true
+++

One of the main challenges we face while developing AI applications is prompt engineering. As developers, we often find ourselves in a cycle of trial and error: writing a prompt, testing it against various inputs, analyzing the outputs, and then tweaking the prompt based on the results. This iterative process requires meticulous tracking of prompt versions, understanding how different phrasings affect model behavior, and maintaining consistency across similar tasks. Managing this complexity becomes even more challenging when we need to chain multiple prompts together or handle different edge cases. This approach, while workable for simple applications, becomes increasingly unsustainable as applications grow in complexity.

This is where DSPy comes in. Created by a team of Stanford researchers, this open-source framework aims to change the way we interact with Large Language Models: instead of explicitly specifying how to achieve the desired outcome, the developers only need to specify what to achieve, letting the model figure out the "how" itself - this is also the core concept of declarative programming. DSPy represents a paradigm shift in how we build AI applications, moving away from manual prompt engineering toward a more systematic programmatic approach. 

In this article, I will first lay the foundation by introducing the core building blocks of DSPy. After that, we will dive into where the real magic happens - Teleprompter, the heart of DSPy's optimization capabilities. 

## core components 

The idea of building AI systems in DSPy is rather straight forward. The process follows three main steps: define the task, evaluate the initial performance, and tune the prompts using Teleprompter. Here's a minimal working example to get started:

```python
# from dspy official documentation
import dspy

# define the LLM
lm = dspy.LM('openai/gpt-4o-mini')
dspy.configure(lm=lm)

# create a simple question-answering predictor
qa = dspy.Predict('question: str -> response: str')
response = qa(question="what are high memory and low memory on linux?")
print(response.response)
```


- a `Predictor` module generates an output from an input based on a given signature

```python
class Predictor(Module):
	def __init__(self):
		self.signature = signature
		self.instructions = instructions
		self.demos = [] # few-shot examples
```

- an `adaptor` transforms the input to a desired format that is suitable for a different kind of module or a step in the pipeline 


```python
class QuestionAnswering(dspy.Module):
	def __init__(self):
		self.generate = dspy.ChainOfThought("question -> answer)

	def forward(self, question):
		return self.generate(question=question)
```

## DSPy optimizers: the self-improving prompt

What makes DSPy truly standout is their automatic prompt optimization system. \<some bridging words\> Once the program and metrics is defined ,



### COPRO (Cooperative Prompt Optimization)

```python
class COPRO(Teleprompter):
	def compile(self, student, trainset):
		# generate initial candidates
		candidates = generate_instruction(breadth)

		for depth in range(max_depth):
			evaluate_candidates()
			refine_best_candidates()
```

key features: 

- breadth-first exploration of prompt state 
- iterative refinement based on performance 
- cooperative learning between iterations 

### MIPROv2 (Multi-stage Instruction Prompt Optimization)

MIPROv2 implements a sophisticated three-stage optimization:

```python
def optimize(self):

	# stage 1: boostrap examples
	demo_candidates = self._bootstrap_fewshot_examples()

	# stage 2: generate instructions
	instruction_candidates = self._propose_instruction()

	# stage 3: Baysian optimization 
	best_program = self._optimize_prompt_parameters()
```

unique characteristics:

- minibtach evaluation for efficiency 
- Bayesian optimization for parameter tuning 
- advanced hyperparameter management 

### Bootstrap few shot 

```python

def bootstrap(self):
	for example in trainset:
		for round in range(max_rounds):
			success = bootstrap_one_example(
				example,
				temperature=0.7 + 0.001 * round
			)
```

features: 

- automatic example generation 
- quality validation through metrics 
- temperature-based exploration 


### Bootstrap few shot with Optuna 

extends on the research paper's findings:

