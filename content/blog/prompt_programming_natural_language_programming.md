+++
title = "Plato's Crutch? Prompt Programming is not Natural Language Programming"
date = 2026-03-08
draft = true
+++

Engineering is often about tradeoffs.
In computer science, algorithm design tends to involve a tradeoff between space and time; and what's called memory hierarchy is the tradeoff between speed and expense.
Likewise with techniques of programming computers: the further these formalizations get from binary machine code, the easier they become to write and reason about, but at the cost of giving up precise control over the instructions executed by the machine.
This programming language hierarchy is not quite so neat as memory hierarchy, because surely the programmer is both less likely to make logic errors or syntax mistakes and more able to control what the computer does when the units of abstraction offer more facility to the programmer.
That a higher-level language like C does not lose tremendous amounts of efficiency relative to writing in assembly is thanks to the compiler, which can translate certain abstract patterns into more efficient, i.e. "optimized," instructions than would be the case were they to be translated more directly.
Indeed, I'm going to be able to write more efficient code using C than assembly thanks to the compiler I use.

Within higher-level languages are more or less abstracted languages.
Python provides dynamically managed memory and built-in data structures like the `list` and `str`, which handle a bunch of complexity.
Rust has both statically allocated arrays and dynamically resizing vectors, and its many types of strings offer more control over where in memory that data is stored and whether it needs to be duplicated.
Above these higher-level programming languages is natural language programming, which has the sentence structure of a language like English.
These sentences compile into the higher-level language in terms of which the NLP language is defined.

Is prompt programming—that is, prompting a LLM to generate code—a kind of natural-language programming?
It certainly feels like it, and to a degree so much more intuitive or indeed _natural_ than existing, non-generative NLP languages.
Yet I am inclined to answer no because ...

In Plato's _Phaedrus_, 

In the story Socrates tells, Thamus, king of Egypt, criticizes the god Theuth for praising writing: "In fact, it will introduce forgetfulness into the soul of those who learn it: they will not practice using their memory because they will put their trust in writing, which is external and depends on signs that belong to others, instead of trying to remember from the inside, completely on their own. You have not discovered a potion for remembering, but for reminding; you provide your students with the appearance of wisdom, not with its reality" (275a).

Ripe for Stieglerian critique:
Someone who knows truth does not attempt to write knowledge down on paper because to do so would be a frivolous pursuit of self-amusement or reminders for old age. Instead, they would commit the knowledge to the memory of another living person. "The dialectician chooses a proper soul and plants and sows within it discourse accompanied by knowledge---discourse capable of helping itself as well as the man who planted it, which is not barren but produces a seed from which more discourse grows in the character of others" (276e-277a). Nathan asked whether such knowledge transfer is possible even with speech.


# Captured on dynalist

One difference this (not being compiled) makes is that it's not clearcut on which side of the code generation the error lies. Well, okay, it's clearly in the generated code, but is that the fault of the prompter or the prompted? With compilation, assuming a correct compiler, the errors come from the programmer who wrote the code being compiled. They aren't introduced by the compiler. (Unless the compiler's own code has an error.) That prompt programming is probabilistic generation rather than unambiguous translation is why I want to situate it adjacent to higher-level programming languages rather than above them in the category of natural-language programming. 

https://sderosiaux.substack.com/p/semantic-closure-why-compilers-know LLMs are not compilers not because of determinism but semantic closure. 

Prompt programming could approach NLP. Based on anecdata, senior engineers have an easier time of getting working code than juniors. Their experience helps them to include more details in the sequence of prompts that would go into planning, implementing, and testing a software feature. In my own limited experience with prompt programming, for example, I needed to be explicit about the abstract data structures to use and how they implement the behavior I expect. This still leaves the concrete implementation details open. For instance, I may specify a circular buffer, but based on the language being generated, the LLM might opt for a hash table or a doubly linked list or an array. 

Kate's distinction between the syntagmatic and paradigmatic in MMWAC may be useful. 
