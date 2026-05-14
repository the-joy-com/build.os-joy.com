# session 1 of building The Joy in the open

So I want to build an agentic system that looks like this ![agentic system schema](./agentic-system.png). I'd like to replace my usage of OpenClaw with something that I've built from the ground up. Currently I'm using OpenClaw via Telegram, so I guess I'm going to spin up a Telegram bot on some compliant bare-metal server located in Europe first. The first brick will be the agentic entry point. I'm going to call my implementation The Joy, because I want it to spark joy in my daily usage of AI. This thing will be _taylor-made_ for both educational and practical reasons.

I will build this iteratively and in an ingenuous way. This is by design.

## setting up a bare metal server

Of course, I managed to lock myself out of my own Ubuntu server twice since it's been ages since I haven't provisioned one. But hey, no pain no gain ![Pain by De La Soul](./pain.jpg). Here, listen to this if you suffer like me with these things; if you forgot how to do it, you need to, in this order:
- change hostname
- create your `sudo`-enabled user
- disable root login (whether with password, SSH, or locally)
- allow for SSH connection
- save the SSH keys you use to connect to your server somewhere safe
- disable password login
- change SSH port

## what landed in code: a Telegram bot that calls a Langgraph graph on receiving input text from a human (me)

Each of these folders is its own Git repo on GitHub under [`the-joy-com` org](https://github.com/the-joy-com):

### [agentic-entry-point](https://github.com/the-joy-com/agentic-entry-point)

- [Initial commit](https://github.com/the-joy-com/agentic-entry-point/commit/c950ac71b20680a4f48a1ab9f5b444e474d30a54) — baseline repo scaffold.
- [Add agentic entry point with Langgraph, env docs, and structured logging](https://github.com/the-joy-com/agentic-entry-point/commit/03c7cd5dc73e2e7016bee1ca526cb88dba31f066) — LangGraph-oriented setup, the runnable entry point and hooks, clearer constants naming, README with run/setup notes, `PYTHONUNBUFFERED` called out in `.env.example`, and `pre_process_hook` output routed through the logger instead of ad hoc prints.

The `pre_process_hook`, that does nothing yet, is IMHO actually the most important part of this system: it will allow me to wire any custom code that I want to run before calling the graph.

### [telegram-bot](https://github.com/the-joy-com/telegram-bot)

- [Initial commit](https://github.com/the-joy-com/telegram-bot/commit/a6da64311e418217fc7133e956eee9e4348c03a5) — baseline repo scaffold.
- [Add Telegram bot with env setup and README](https://github.com/the-joy-com/telegram-bot/commit/61c87b9001b27560f9bf7cc5f2bfaabacce5d27d) — bot implementation plus supporting pieces: constants, `requirements.txt`, `.env.example`, `.gitignore`, and a README documenting setup and how to run it locally.

Together, that is the first slice of replacing the OpenClaw-over-Telegram flow: a dedicated Telegram bot repo and an agentic entry point repo you can wire up or run as a service next. Then after that, I'm thinking of:
- selecting the model based on the input
- adding a Redis memory checkpointer wired to the Telegram session id to keep continuity in my conversations
- deal with context rot by summarizing the conversation accordingly, and of course do that based on the model that has currently been auto selected