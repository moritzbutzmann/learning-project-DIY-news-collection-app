Requirements for an AI News Collecting app:

- I want to have an AI News collector which scrapes a set of configurable sites in the internet and collects news articles from them
- The scrape should be triggered via a button on the UI & there should be an option to watch the ongoing scrape (keep it !VERY simple)
- I want to have some kind of editor functionality where I can press a button and a draft for the newsletter is being generated, I want to be able to edit the newsletter
- the draft should include the top 10 news articles which have been collected
- each article should be scored on relevance based on a configurable set of topics by an LLM
- the LLM to be integrated is anthropic haiku-4.5 using a custom endpoint and api key, please use litellm for this
- please configure as a default source https://news.ycombinator.com/rss and set as some default topics around ai for coding