## Pre-Face (introductory talk):

- the goal is for you to build a DIY AI News Collector: the only must-have criteria are:
  - must have a WebPage / Frontend with a UI
  - must integrate an LLM
  - must have some kind of backend & use a database as storage
  - must be able to autonomously scrape websites & provide a weekly summary
- keep in mind: for the AI-Newscollector, we've spend in total around 3+ 40hr weeks
  - -> during this time, we learned a lot, it could be that now we are a bit faster
- FYI: you can find the full version [here](https://ai-news-collector.sandbox.starplatform.cloud/login), if you want to try it out afterwards
- Anyway: the goal we are using is that everyone of you has his own version of a DIY AI News Collector, but what I will try to show you, is how it feels using Claude Code for doing this:
  - how does working with claude code look like?
  - what Claude can or cannot do? Where are its limits at the moment?
  - what kind of patterns are beginning to emerge (e.g. skills, full-round-trip)
- The ambition of this document is to contain the minimal amount of instructions in order to be able to build your own DIY ai news collector in 1.5 hours. There are many things which one could improve in the prompts and workflows, they are minimal on purpose. If you think, some things could or should be improved, please do so. Experiment with it.


### 0. Introduction & Minimal Env-Setup (5min)

- Speaker giving an introduction to the DIY project
- open this project inside an isolated environment (devcontainer for macos is included)
  -> TODO: to be clarified with colleagues, but in vs code: open in devcontainer works on mac here
- this file is excluded via .claudeignore from the files claude is seeing to prevent any confusion during implementation
- start claude in the terminal of VS Code using the command `claude`
- make sure to use opus 4.6 with medium reasoning, you can check so with entering `/model` into claude
- set up claude.md
  - claude.md are the core agent instructions, if you want to change any processes or behavior of claude, pls put it in there.
  - E.g. Prompt: `Please add to the claude.md file to always note bugs you found or ideas for improvements which are not yet scoped to be implemented in the TODO.md file. This is your living backlog which is maintained in accordance with human input`
  - approve the things claude is asking to do, that is, how claude is normally working
  - -> if you have an idea, you can just ask claude to add it to the todo.md file and later when you finished a task, you can come back to this file and ask claude to implement items from it or to refine the backlog