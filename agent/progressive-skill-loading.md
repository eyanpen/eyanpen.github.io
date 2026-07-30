# How an Agent "Progressively" Loads a Skill and Calls Tools — A Story Told Through a Real Request Log

> We often hear a claim: give an Agent hundreds of tools, and it actually gets dumber, slower, and more expensive.
>
> So how do those mature Agents actually solve this problem? One of the answers is called **progressive disclosure**.
>
> This article starts from a real LLM request log and peels it apart layer by layer: how an Agent pulls the full description of a Skill into its context only "when it's needed," and how it goes on to execute the tools mentioned inside that Skill. Once we've walked through this concrete example, we'll talk about other mainstream approaches in the industry, and how all of this relates to Function Calling.

---

## 1. First, What Actually Happened in the Log

The log records a scheduled job: an Agent named Hermes is woken up by cron at 6 a.m., with the task of "using the mx-search skill to grab the day's important financial news and put together a morning briefing."

The most valuable part of the log is the `messages` array — it fully preserves what the Agent did, step by step, in this round of conversation. Let's pull out its sequence of actions:

1. In the system prompt (system), only the **name and one-line description** of each Skill are listed, including skill_view (built-in), mx-search, and so on;
2. The user message (user) gives the task;
3. The assistant first thinks for a moment: "this job is related to mx-search," and then calls `skill_view(name='mx-search')`;
4. The tool returns the **full manual** of mx-search (how to call it, what it returns, what to do on errors);
5. The assistant then uses `execute_code` to write Python, first figuring out what the scripts bundled with this Skill look like;
6. It uses `execute_code` again to actually launch the search and process the data;
7. Finally it puts together the morning briefing.

This "shallow-to-deep, load-on-demand" process is a textbook demonstration of progressive disclosure. Let's go through it layer by layer.

---

## 2. Layer One: The System Prompt Holds Only the "Table of Contents," Not the "Body Text"

Open the system message in the log, and you'll see a passage like this:

```text
<available_skills>
  ......
  mx-search:
    - mx-search: This skill is based on Eastmoney's Miaoxiang search capability, performing intelligent source filtering for financial scenarios...
  ......
</available_skills>
```

Notice that here **each Skill has only a name and a one-line description** — no details at all about how exactly to use it, which interface to call, or what fields it returns.

The design of this layer is crucial. We can compare it to the **table of contents** of a book: the table of contents only tells us "Chapter 3 is about financial search," but it won't copy the entire body of Chapter 3 onto the title page. Why do it this way? Because the body text is long. The full manual of mx-search (which we'll see in the next section) runs to several thousand words. If we stuffed the full manuals of every Skill in the system into the system prompt, there would be two fatal problems:

- **Expensive.** This content gets re-sent to the model in **every single** request, and the token cost balloons linearly with the number of Skills. Installing 100 Skills is like reciting 100 manuals before every sentence you say.
- **Dumb.** Cramming too much irrelevant information into the context dilutes the model's attention, making it more likely to pick the wrong tool or focus on the wrong thing. This is a phenomenon backed by experiments — the more tools/information there are, the more selection accuracy drops.

So the system prompt holds only the table of contents. At the same time, it also writes a mandatory instruction telling the model how to use this table of contents:

> Before replying, scan the Skill list below. As long as any of them is relevant, you must use skill_view to load it in, and follow it.

In short: **keep the table of contents on you at all times, and flip to the body text only when you need it.**

---

## 3. Layer Two: Only After a Hit Do We Pull in the "Body Text"

After reading the task, the Agent determines that this job is related to mx-search, so it issues the first tool call:

```json
{
  "name": "skill_view",
  "arguments": "{\"name\":\"mx-search\"}"
}
```

What this step returns is much richer. It brings back the entire `SKILL.md` of mx-search, including:

- The functionality description (what it does, what scenarios it fits);
- The specific way to call it (command examples like `python ./mx_search.py "latest research reports on Kweichow Moutai"`);
- Notes for scheduled jobs (how to read the API key, and that you must `print` the output or it will be judged as silent);
- A reference table of "exception situations and how to handle them";
- A **fallback plan** for when the API is unavailable (using Eastmoney's public news-flash interface as a backup);
- The meaning of the returned fields (what `title`, `secuList`, and `trunk` each are).

Besides the body text, the returned result also contains a few very interesting fields:

```json
{
  "linked_files": { "scripts": ["scripts/mx_search.py"] },
  "usage_hint": "To view linked files, call skill_view(name, file_path)...",
  "readiness_status": "available",
  "setup_needed": false,
  "missing_required_environment_variables": []
}
```

This shows that loading a Skill isn't just "reading a piece of text." At runtime it also does two things for the Agent along the way:

1. **Tells it what deeper resources are still there to dig into** (`linked_files` points out there's also an `mx_search.py` script, and `usage_hint` explains that if you want to look at it, just call `skill_view` one more time);
2. **Does an availability health-check in advance** (this Skill needs the `MX_APIKEY` environment variable; after checking at runtime it reports `readiness_status: available` with an empty list of missing variables, meaning "the environment is all set, ready to use directly").

This is **Layer Two** of progressive disclosure: only when a Skill is actually hit does its several-thousand-word body text get loaded into the context. For a Skill that never gets used, its body text will never occupy a single token.

---

## 4. Layer Three: Deeper Scripts and Reference Material Can Still Be Pulled On Demand

The `linked_files` returned in Layer Two actually plants an entry point to a "Layer Three."

mx-search comes bundled with a `scripts/mx_search.py`. The Agent can entirely choose to call `skill_view(name='mx-search', file_path='scripts/mx_search.py')` one more time to read the script's source code into context as well.

The significance of this layer is that **resources can be placed arbitrarily deep, but the loading depth is decided by the Agent based on actual need.** Some tasks are fine with just a glance at the table of contents; some need to read the full body text; and only a few that actually roll up their sleeves will go deep into the script details. This is a tree that "expands on demand," not a big pancake spread out all at once.

We can summarize the three layers in one sentence:

- Layer One — **name + one line**, always in the system prompt, cost extremely low;
- Layer Two — **the full manual**, loaded only on a hit;
- Layer Three — **scripts / reference files / templates**, dived into only before rolling up your sleeves.

---

## 5. How Do the Tools Mentioned in a Skill Actually Get Called?

This is where many people get confused. We might assume: mx-search is a Skill, so is its internal search capability also a registered "tool" that the model can just click to use?

**No, it isn't.** This log makes it very clear.

The mx-search Skill itself is **not** registered as a callable function in the request's `tools` list. It's just "a manual + a Python script." So how does the Agent use it? The answer is: with the help of a **general-purpose code execution tool**, `execute_code`.

We can understand the whole chain like this:

1. The **table of contents** (available_skills in the system prompt) tells the Agent "what capabilities are on the menu";
2. The **manual** (the body text pulled back by skill_view) tells the Agent "how to use this capability";
3. The **general-purpose execution tool** (execute_code / terminal) lets the Agent "actually get the job done by following the manual."

The Skill is responsible for "knowing how to do it," and the general-purpose tool is responsible for "actually doing it." The division of labor is clear.

---

## 6. Why Go Through All This Trouble?

Some might ask: wouldn't it be simpler to just register every capability as a tool and let the model pick?

When there are few capabilities, that's indeed the case. But as capabilities grow, the direct-registration approach collapses along three dimensions at once:

- **The cost dimension**: every tool's JSON schema has to be sent to the model in every round of the request. With hundreds of tools, the descriptions alone can eat up tens of thousands of tokens — and you **pay for them again with every sentence**.
- **The accuracy dimension**: the more options there are, the more likely the model is to pick wrong. Progressive disclosure lets the model face only a handful of genuinely relevant options each time.
- **The maintenance dimension**: capabilities exist as files (Skills), so updating one capability just means editing one Markdown file — no need to touch the Agent's code, and no need to re-register a pile of schemas. Those `skill_manage` and `skill_view` tools in the log are precisely for managing these files.

In one sentence: **progressive disclosure is essentially about decoupling "the number of capabilities" from "the overhead of each round of request."** Installing 1,000 Skills and installing 10 Skills can result in almost the same base cost for a single round of request.

---
## 7. What Exactly Is Its Relationship to Function Calling?

Once we get this straight, everything before it ties together.

**Function Calling is the "underlying mechanism," and progressive disclosure is the "upper-layer strategy."** They're not on the same level, and they're not an either-or choice.

We only need to look at the `tools` array in this log to understand. What's registered in there is actually a batch of **general-purpose meta-tools**: `skill_view`, `skills_list`, `skill_manage`, `execute_code`, `terminal`, `delegate_task`, a bunch of `browser_*`, and so on — about twenty in total. And those real domain capabilities — mx-search, cronjob-best-practices, finance-news-briefing-generator — are **not a single one of them in this array**. They exist as files and are injected on demand as text.

So the relationship becomes clear:

- **Function Calling is the model's only primitive for "triggering an action."** No matter what the model wants to do, in the end it has to emit a structured tool_call (function name + JSON arguments), which the runtime executes and feeds the result back. In the log, the calls to `skill_view` and `execute_code` are, at their core, standard Function Calling.
- **Skills / progressive disclosure are a layer of packaging riding on top of Function Calling.** Instead of registering a function for every capability, it registers only a handful of general-purpose functions (read a Skill, execute code, spawn a sub-Agent), and "dimensionally reduces" hundreds or thousands of concrete capabilities into **text and files** that can be loaded on demand.

We can drive the point home with a comparison:

- **The naive pure-Function-Calling approach**: one capability = one registered function schema. N capabilities means N schemas lying **forever** in every round of request. Once the number grows large, it's expensive, dumb, and hard to maintain.
- **The progressive-disclosure approach**: register only a handful of general-purpose functions (among which are the two master keys, "load a certain capability's description" and "execute code"), and externalize all concrete capabilities into files, loaded only when needed and landed through general-purpose functions when working.

So **progressive disclosure isn't meant to replace Function Calling, but to cure the disease of "Function Calling with too many tools."** Function Calling is still that pipe — it's just that we no longer pour all the water in at once; instead, whichever stream we need, that's the pipe we connect.

---

Back to that log at the very beginning, it actually gives a complete demonstration of a mature Agent's capability-management philosophy:

- The system prompt holds only the **table of contents** (name + one line), cost extremely low, always online;
- When the task hits a certain Skill, only then does it use `skill_view` to pull in the **full manual**;
- Only when it's truly time to roll up the sleeves does it go deep into the **script details**, and it may even use `execute_code` to run code directly in a sandbox and process large data, bringing back only the result into the context;
- And the underlying primitive supporting all of this is, from beginning to end, those few general-purpose **Function Calling tools**.

The essence of progressive disclosure can be captured in one sentence: **separate "what I can do" from "what I need to do right now" — the former can be unlimited, the latter always stays lean.** This is also why those Agents that can mount a massive number of capabilities still stay fast and accurate in a single round of conversation.

---

If you found this article helpful, feel free to **like, bookmark, and follow**. I'll keep sharing more valuable content. Your support is the greatest motivation for my writing!
