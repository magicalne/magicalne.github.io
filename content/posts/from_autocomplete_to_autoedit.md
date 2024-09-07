---
layout: post
title: "From Autocomplete to Autoedit: Is Cursor All Hype?"
date: 2024-09-07
tags:
- llm
- editor
categories:
- insight
---

**tl;dr:** We need an open-source LLM that can predict the next edit instead of autocompletion.

# Cursor vs Zed

Cursor, made by Anysphere, is an AI-first startup aiming to revolutionize coding with AI. The editor is just a way to verify their theory.

Zed, created by the former team of Atom, is a GPU-accelerated editor that incorporates AI as one of its features.

Zed is a good editor, and Cursor is currently the best AI copilot available.

# Zed's Approach

- Chatting with LLM in the sidebar, which is a special editor.
- Slash commands make it easy to set up context in the sidebar.
- Inline assistant: insert or rewrite code. It can read the whole context in the sidebar.
- Workflow: generate suggestions and apply suggestions one by one.

I like the "sidebar and slash commands" idea. That's why I copied this idea and created a Neovim plugin called [nvim.ai](https://github.com/magicalne/nvim.ai).

However, I don't like how Zed implements the `workflow`. Inspecting the prompt from [edit_workflow.hbs](https://github.com/zed-industries/zed/blob/main/assets/prompts/edit_workflow.hbs), it is hardcoded with Rust examples, which means that **it only supports Rust**.

Looking at how Zed processes steps in the prompt from [step_resolution.hbs](https://github.com/zed-industries/zed/blob/main/assets/prompts/step_resolution.hbs), the step part is the most tricky one in the whole copilot, not just Zed, but all of them.

The `step resolution` aims to find the position to update in the code. Zed takes advantage of LSP. It makes LLM return the position of the `symbol` instead of line numbers. This is pretty accurate for LLMs like Sonnet3.5 and GPT-4.

# Cursor's Approach

Cursor goes way further. I want to talk about two major features: Tab flow and Instant Apply.

## Tab Flow

Cursor solves the problem that GitHub Copilot doesn't. In their blog, they said: "Developers are not inserting code all the time. Developers are editing most of the time." So they trained a model or two, likely with 3b-14b parameters, that predicts the next `edit`, instead of generating code. The data structure of the prediction looks like:

- type: insert/update/delete?
- position, or positions
- content

This is absolutely brilliant. And that makes me believe `Anysphere` is an AI company. They collect a lot of data, codebases, and they will collect more.

## Instant Apply

I think Zed took the `workflow` idea from `Instant Apply`. But it's not as good as Cursor's, in my opinion. There is no `workflow` feature in Cursor. But you can `apply` the code in the sidebar, I mean any code. Remember I said the `step resolution` is the most tricky part? Let's see how Cursor resolves this problem.

Cursor explains `instant apply` [in the blog](https://www.cursor.com/blog/instant-apply). The secret is fine-tuning and rewriting the entire file. It looks like Cursor wants to provide the best service, and at the same time, it doesn't want to rely on the best closed-source LLMs like GPT-4 and Sonnet3.5. The `instant apply` feature should be fast and stable. And of course, Sonnet 3.5 can solve the problem. But rewriting the entire file? That means a lot of tokens. And LLMs tend to be lazy on redundant content. LLMs don't like to repeat content that appears before. They prefer to be lazy with `// some unchanged code here...`. So fine-tuning is a good approach. And clearly, Cursor made it.

### But Why Rewrite the Entire File?

At least you don't count on the capability of LLMs to generate the right line numbers of where to edit the code. I think Zed's approach is better.

# Conclusion

So, is Cursor all hype? Kind of, but not all of it. I like the idea that they trained the model to predict the next edit. And I like that they fine-tuned a model to work on instant apply. I hope some company can train a similar model so that I can add `tab flow` to my Neovim plugin. And about `Instant Apply`, it's a fancy feature, but not a `must` for me.

Just like the old saying, LLMs make people who are not good at some field do better, make people who are okay at some field do a little better, but make people who are good at some field do nothing.
