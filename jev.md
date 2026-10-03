# JEV

created by [typesafe.ai](https://typesafe.ai) 

CEO: [Diogo Almeida](https://www.linkedin.com/in/diogomda/) | CTO: [Erik Gafni](https://www.linkedin.com/in/erik-spock-gafni-906b0125/) | COO: [Sasha Sheng](https://www.linkedin.com/in/sashasheng/)

## Notes 1

![ticket_routing](/assets/jev_assets/ticket_routing.png)

**Scenario: support inbox**
- 2,000 tickets every morning → each must reach the right team
    - billing → billing team | bug → engineering | pricing → sales
- Can't read by hand → write code to sort → most teams reach for an **LLM**

<table>
        <tr>
            <td><img src="/assets/jev_assets/ticket_routing_LLM_1.png" height="400" width="600" alt="ticket_routing_LLM_1"></td>
            <td><img src="/assets/jev_assets/ticket_routing_LLM_2.png" height="400" width="600" alt="ticket_routing_LLM_2"></td>
        </tr>
</table>

**Problem 1: LLM answers a sentence, code needs a label**
- Prompt = plain-English team descriptions + ticket → sensible answer, **no labeling, no training**
- Reply is a *sentence* ("primarily billing, though there is a technical element")
    - Fine for a human, useless for code → `if` can't branch on a sentence
    - Can't keyword-search "billing" → "technical" is in the same sentence

![ticket_routing_schema](/assets/jev_assets/ticket_routing_schema.png)

**Fix: schema**
- Give the model a schema: `department` ∈ {billing, technical, sales}
- Output = bare word (e.g. `billing`) → code can act on it

**Problem 2: label only, no certainty**
- You get a label per ticket → can't tell a **clear pick** from a **borderline** one

![ticket_routing_confidence](/assets/jev_assets/ticket_routing_confidence.png)

- Example
    - "Upgraded but no features" → labeled billing, but could be technical (payment failed **or** payment OK but feature not switched on)
    - "Double charge" → clearly billing
    - LLM returns the **same single word** for both
- Of 2,000 tickets → can't separate "100% sure" from "needs human review"

**Patch: ask for a confidence score**
- Extra call / extra `confidence` field → returns e.g. 0.9
- Open question → **how reliable is that number?**

**LLMs vs Jev (Training)**

<table>
        <tr>
            <td><img src="/assets/jev_assets/training_LLM_vs_Jev_1.png" height="400" width="600" alt="training_LLM_vs_Jev_1"></td>
            <td><img src="/assets/jev_assets/training_LLM_vs_Jev_2.png" height="400" width="600" alt="training_LLM_vs_Jev_2"></td>
        </tr>
        <tr align="center">
            <td> Reinforcement Learning from Human Feedback (RLHF) </td>
            <td> Reinforcement Learning for Calibrated Decisions (RLCD) </td>
        </tr>
</table>

- **How LLMs are trained (nearly all major ones)**
    1. **Pre-training** → huge text corpus (blogs, articles, videos, Wikipedia, forums) → predict next word
    2. **RLHF** → humans rate answers → model tuned toward highly-rated answers
        - Human feedback = "did a person *like* it?"
- Gap → training **never compared the model's confidence number to the actual correct answer**
- **Jev (typesafe.ai)** fixes this
    - Probability is **tuned against how often the answer is actually right**
    - Method = **RLCD**
    - **No human raters** → generates its own training data → already knows the right answer → scores whether the number matched reality

**System One vs System Two**

- Inspiration: *Thinking, Fast and Slow* (Daniel Kahneman) → 2 ways people think
    1. **System One = fast**, answer just arrives
        - e.g. recognising a friend's face → name comes instantly
    2. **System Two = slow**, effortful, step by step
        - e.g. 17 × 24 → stop, hold numbers in head, work through steps

<table>
        <tr>
            <td><img src="/assets/jev_assets/system_one.png" height="400" width="600" alt="system_one"></td>
            <td><img src="/assets/jev_assets/system_two.png" height="400" width="600" alt="system_two"></td>
        </tr>
        <tr align="center">
            <td> Jev → System One </td>
            <td> LLM → System Two </td>
        </tr>
</table>

- Most LLMs = **System Two** → consumer is a **human** (e.g. ChatGPT chat)
    - For **app code** making quick AI calls → too **costly**, too **slow**, **unreliable**
    - Few LLMs built primarily for code as consumer → **Jev fills this gap**
- **System One (typesafe's use of the term)** = class of models where **code is the primary consumer**, not a person → Jev's category

**How to call Jev**
- Not another prompt → send **two things**
    1. **State** = whatever you want judged
        - e.g. ticket: *"I upgraded to pro but the new features have not appeared in my account."*
    2. **Questions** about it + criteria (billing / technical / sales…)
        - Many questions in **one run** → all answered together
- **3 question types**
    - `choice()` → pick **one** from a declared list
        - e.g. department: billing / technical / sales
    - `score()` → place on an **ordered scale** you define
        - e.g. frustration: calm < annoying < furious
    - `noul()` → plain **yes/no statement** → one number **0–1**
        - e.g. "customer is asking for money back" → 0.9 = almost certainly a refund request

**Output: probability spread over your options**
```
{
    "choice": "technical",
    "probabilities": {
        "billing": 0.38
        "technical": 0.62
        "sales": 0
    }
}
```
- Example ticket → billing 0.38, technical 0.62 → **not 100% sure**
- **Calibration** → answers scored 0.8 should be right ~80% of the time
- Probabilities **always sum to 1** → model splits one whole across your options
- **Confidence** (second value returned) = **how evenly the whole is split**
    - One option takes nearly all → **high** confidence
    - 2–3 options share almost equally → **low** confidence

**Confidence gate (what to do with the number)**
- **≈ 0.9+** → **act** (route it, no human ever looks)
- **0.5 – 0.9** → **confirm first**
- **< 0.5** → **send to human** or a bigger LLM that "thinks harder"

![ticket_routing_jev](/assets/jev_assets/ticket_routing_jev.png)

- Back to the 2,000 tickets → each has a probability split + confidence
    - High → act | Medium → confirm | Low → person / bigger LLM

**What Does This Actually Cost?**

<table>
        <tr>
            <td><img src="/assets/jev_assets/step_one.png" height="400" width="600" alt="step_one"></td>
            <td><img src="/assets/jev_assets/step_two.png" height="400" width="600" alt="step_two"></td>
        </tr>
        <tr align="center">
            <td> Step One </td>
            <td> Step Two </td>
        </tr>
</table>

- **Tokens** = short pieces of words → you pay for **input and output**
- Example: **gpt-4o-mini** → ~$0.15 / 1M tokens in, ~$0.60 / 1M out → **output ≈ 4× input**
    - Ratio holds industry-wide → **generating text is the expensive part**
- **Why generation is costly (autoregressive loop)**
    - Load whole model from GPU memory → cores → process → **1 token** → feed token back in → reload weights → repeat…
    - Hammers **memory bandwidth** → the biggest bottleneck / scalability limit of AI infra
- **Jev pricing**
    - **$42 / 1B tokens ≈ $0.042 / 1M tokens**
    - **Output tokens are FREE** → model never writes text, just returns a number + probabilities
    - All questions answered in **one single pass** → 10 questions ≈ cost of 1 → **changes how you write code**

```
questions = {
    "department": Choice("Which team should handle this"),
    "frustration": Score("How frustrated do they sound"),
    "churn_risk": Noul("They are about to cancel"),
    "mentions_competitor": Noul("They name a competitor"),
    "is_urgent": Noul("This needs to jump the queue"),
    "refund_request": Noul("They are asking for money back"),
    "is_spam": Noul("This is not a real ticket"),
    "language": Choice("Which language is it written in"),
    "security_issue": Noul("They report a security problem"),
    "feature_request": Noul("They ask for something new")
}
```

- **Multiple questions, one call, one pass**
- Upgrade ticket → department never landed (technical 0.62 / billing 0.38) → **confidence 0.42 < 0.5** → goes to a human anyway
    - But **which** person, **how soon**? → the **other questions** decide
    - Angry + talking cancel → **senior person today**
    - Named a competitor → **copy to sales**
    - Calm, nothing on fire → human, but **can wait its turn**

```
if frustration > 1.5 and churn_risk > 0.7:
    route_to_senior(ticket)
elif mentions_competitor > 0.8:
    copy_to_sales(ticket)
else:
    queue_for_human(ticket)
```
![speculation_fanout](/assets/jev_assets/speculation_fanout.png)

- **Speculation fanout** (docs term)
    - **Speculative** → ask things you may never use
    - **Fanout** → all branch off **one call**

**Good Fit vs Bad Fit**

<table>
        <tr>
            <td><img src="/assets/jev_assets/jev_good.png" height="400" width="600" alt="jev_good"></td>
            <td><img src="/assets/jev_assets/jev_bad.png" height="400" width="600" alt="jev bad"></td>
        </tr>
        <tr align="center">
            <td> Jev → Good Fit </td>
            <td> Jev → Bad Fit </td>
        </tr>
</table>

- typesafe lists **4 good-fit patterns** on its website
    - Ticket agent built here = **confidence-gated routing**
    - Multiple questions in one call = **speculative fanout**

---

## Notes 2

![diogo_almeida_tweet_1](/assets/jev_assets/diogo_almeida_tweet_1.png)

- [jev](https://x.com/NathanFlurry/status/2100036101809619314) doesn't replace gpt/claude
- jev is just a really smart switch statement
    - if 2016 ML classifiers got 2026 levels of intelligence
- new type of tool → makes many workloads insanely **fast, cheap, accurate**
- **System One** model → no deep thinking, impromptu, fast decision
- needs a **predefined set of options** → tells you which one to take
- it cannot:
    - write code
    - generate natural language
    - reason step by step / show its work
    - produce any output you didnt define in advance
    - pick from more than ~255 options in one-shot
- but it can:
    - classify, route, score and rank
    - give confidence
    - pick the right branch, tool, model, or sub-agent
    - judge/verify/guardrail an LLM's output
    - label tons of rows

- agentic workflows = lots of **decision-making**
    - e.g. Claude Code building a project → thinks, decides, checks output, multiple turns → many decisions
    - Jev makes those decisions → whole workflow **faster + cheaper**
- common workflow shape:
    - LLM proposes options --> jev decides --> code executes (fits well with Claude Code and MCP)

![jev_use_case_1](/assets/jev_assets/jev_use_case_1.png)

**Example: insurance claim triage**
- Goal: automate processing of incoming claims
- 3 things to identify
    1. **Fraud risk** → fraud or not?
    2. **Claim type** → vehicle / health / property
    3. **Severity** → how much money the insurer will spend
- Agent combines signals → picks route
    - **auto-process** | **senior review** | **reject as fraud**

**LLM**
```
from langchain.chat_models import init_chat_model
from pydantic import BaseModel, Field

STATE = ("Customer reports their car was rear-ended at a red light yesterday."
         "Police report filed. Requesting repair estimate coverage of $4,200.")

class ClaimTriage(BaseModel):
    fraud_risk bool = Field(description="True if the claim shows signs of being fraudulent or exaggerated")
    claim_type: Literal["auto", "property", "health"] = Field(description="Which policy line this claim belongs to?")
    severity: Literal["minor", "moderate", "major"] = Field(description="How severe the claim loss is?")

llm = init_chat_model("groq:openai/gpt-oss-120b")
structured_llm = llm.with_structured_output(ClaimTriage)

PROMPT = f"""Analyze this insurance claim: fraud_risk (true/false), claim_type
(auto/property/health), severity (minor, moderate, major).

Claim: {STATE}"""

@timed
def run_llm():
    return structured_llm.invoke(PROMPT)

llm_result, llm_ms = run_llm()
llm_result
```

```
run_llm: 756.7 ms
ClaimTriage(fraud_risk=False, claim_type='auto', severity='moderate')
```

- Usual AI-engineer approach → define structured schema → LLM fills it
    - Result: fraud=False, type=auto, severity=moderate
- Looks solid, but **signal is missing**
    - Confidence in fraud call? → unknown
    - Severity "moderate" on what basis? → undefined unless spelled out in prompt
- Key insight → problem has **well-typed, well-scoped output** with **fixed options**
    - 2 are confidence-style scores, 1 is a choice from a fixed set
    - Like **softmax** → gives probability to each output

**How Jev does it**
- Used **LangChain integration** for typesafe
- **State** = the data (user prompt / claim)
- **Questions** = the task
    - Fraud → `noul` (yes/no primitive)
    - Claim type → `choice` (like a dropdown: auto / property / health)
    - Severity → `score`
        - Minor < $1,000 | Moderate $1,000–$10,000 | Major > $10,000

**Jev**
```
from langchain_typesafe import Choice, Noul, Score, TypeSafeClassifier

classifier = TypeSafeClassifier()

@timed
def run_jev():
    return classifier.invoke({
        "state": STATE,
        "questions": {
            "fraud_risk": Noul(instructions="Does this claim show signs of being fraudulent or exaggerated?"),
            "claim_type": Choice(
                instructions="Which policy line does this claim belong to?",
                criteria={
                    "auto": "vehicle collisions, accidents, and auto damage.",
                    "property": "home, theft, fire, or property damage.",
                    "health": "medical treatment or injury-related claims."
                }
            ),
            "severity": Score(
                instructions="How severe is the claimed loss?",
                criteria=["Minor, under $1,000.", "Moderate, $1,000-$10,000.", "Major, over $10,000]
            )
        }
    })

response, jev_ms = run_jev()
response
```

```
run_jev: 373.3 ms
ClassifierResponse(model='jev-1.13.0', answers={'fraud_risk': NoulAnswer(type='noul', noul=0.17), 'claim_type': ChoiceAnswer(type='choice', choice='auto', probabilities={'property': 0.0, 'auto': 1.0, 'health': 0.0}, confidence=1.0), 'severity': ScoreAnswer(type='score', score=1.0, legend={0: 'Minor, under $1,000.', 1: 'Moderate, $1,000-$10,000', 2: 'Major, over $10,000.'}, probabilities={0: 0.0, 1: 1.0, 2: 0.0}, confidence=1.0)}, usage=Usage(input_tokens=466, output_tokens=72), request_id='req_01a8c9e8a57275b5a21ec5e8a3fcea15')
```

- **Reading the result**
    - Fraud risk **17%** → low, strong signal
    - Claim type **auto**, confidence 1.0, with full distribution
    - Distribution ≈ **softmax** (like old neural-net classifier output layer) → but **general** and **understands semantics**
- **Noul vs Score**
    - Score → **categorised** level (0, 1 or 2)
    - Noul → always a **probability 0–1**

```
fraud = response.nouls["fraud_risk"]
claim_type = response.choices["claim_type"]
severity = responses.scores["severity"]

print("Claim Triage Result")
print("=" * 40)

print(f"{'Fraud Risk:':<16} {fraud.noul:.0%}")
print(f"{'Claim Type:':<16} {claim_type.choice} (confidence {claim_type.confidence:.0%})")
print(f"{'distribution:':<16} " + ", .join(f"{k}={v:.0%}" for k, v in claim_type.probabilities.items()))
print(f"{'Severity:':<16} {severity.score:.2f} ({severity.legend[round(severity.score)]})")

print("=" * 40)
```

```
Claim Triage Result
========================================
Fraud Risk:     17%
Claim Type:     auto (confidence 100%)
distribution:   property=0%, auto=100%, health=0%
Severity:       1.00 (Moderate, $1,000-$10,000)
========================================
```

- **Speed + cost**
    - LLM *could* do this too → difference is **speed** and **cost**
    - Here only **2× faster** (Colab, tiny GPT-OSS model)
    - vs models used in real agents (Opus, Astra etc.) → **70–80× faster**
    - **Token cost** → Groq free API here; in production not free → Jev = **fraction of a dollar** vs Opus/Astra
    - **Output token cost = 0** → founder tweeted they didn't charge as it was insignificant
        - LLMs: output costs **more** than input
        - Jev: most content lives in the **input**; output is just a probability distribution

```
print(f"Jev: {jev_ms:.1f} ms")
print(f"LLM: {llm_ms:.1f} ms")
print(f"Jev was {llm_ms / jev_ms:.1f}x faster")
```

```
Jev: 373.3 ms
LLM: 756.7 ms
Jev was 2.0x faster
```

- ⚠️ **Hallucination not ~0**
    - Guardrail use case → failed a couple of times, worked a couple of times
    - Depends on the developer → take **same cautions as with LLMs**

### Use Cases of Jev

**1. Dynamic Model Routing**
![jev_use_case_2](/assets/jev_assets/jev_use_case_2.png)

- Common real-world task → choose **cheaper/faster** vs **more powerful** model
    - Earlier: semantic routing or an LLM → Jev can do it
- **LangChain provides a Jev-based model router**
    - You specify **criteria** per model → it auto-chose the powerful one for the complex query
- Is Jev an LLM variant?
    - Yes, a **language model** (knows language) but **not** trained with typical RLHF / RLVR
    - Uses a **new RL paradigm** + **new model architecture**

<table>
        <tr>
            <td><img src="/assets/jev_assets/new_RL_paradigm_RLCD_1.png" height="400" width="600" alt="new_RL_paradigm_RLCD_1"></td>
            <td><img src="/assets/jev_assets/new_RL_architecture.png" height="400" width="600" alt="new_RL_architecture"></td>
        </tr>
        <tr align="center">
            <td> New RL paradigm → Reinforcement Learning for Calibrated Decisions (RLCD) </td>
            <td> New RL Architecture → Auto-Regressive Token Generation ❌ | Parallel Batch Probability Distribution ✅ </td>
        </tr>
</table>

```
from langchain.agents import create_agent
from langchain_typesafe.experimental.middleware import ModelChoice, ModelRouterMiddleware

router = ModelRouterMiddleware(
    choices={
        "fast": ModelChoice(model="groq:openai/gpt-oss-20b",
            criteria="Order status lookups, FAQ answers, and simple account questions with a clear, direct answer."),
        "powerful": ModelChoice(model="groq:openai/gpt-oss-120b",
            criteria="Billing disputes, refund exceptions, policy edge cases, and anything involving customer frustration or financial risk.")    
    },
    instructions="Choose the least costly model that can resolve the ticket safely and correctly."
)

agent = create_agent("groq:openai/gpt-oss-20b", middleware=[router])

result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": "My order #A-4821 says delivered but I never received it and this is the third time this month. I want a refund and I'm considering canceling my subscription."
    }]
})

route = result["model_route"]
reply = result["messages"][-1].content

print("Support Ticket Result")
print("=" * 50)
print(f"{'Routed to:':<12} {route.choice} (confidence{route.confidence:.0%})")
print("=" * 50)
print("Agent response:\n")
print(reply)
```

```
Support Ticket Result
==================================================
Routed to: powerful (confidence 88%)
==================================================
Agent response:

I'm really sorry to hear that your order #A-4821 never arrived-especially after a couple of similar issues this month. Let's get this sorted 

**What I'll do for you**

1. **Open an investigation** with our shipping partner to locate the package and determine what went wrong.
2. **Process a full refund** for the missing order as soon as investigation is confirmed.
3. **Review your subscription** to see if there's a better plan for you, and if you still wish to cancel, I can handle that instantly.

**What I'll need from you**

- The **email address** associated with the order (so I can pull up the full order record.)
- The **shipping address** the order was sent to (just the street, city, and zip-no need for full credit-card details).
- The **date** you expected delivery (or the tracking number, if you have it handy).

**Next Steps**

Once I have the information above, I'll:

- File the missing-package claim and keep you posted on any updates (usually within 24-48 hours).
- Issue the refund to the original payment method; you'll receive a confirmation email as soon as it's processed.
- Cancel or adjust your subscription per your request, and I'll send a final confirmation of that change.

Again, I'm truly sorry for the inconvenience. Let me know the details when you can, and I'll get everything moving right away. If there's anything else I can do, please let me know. Have a great rest of your day.
```

- Trivia: Diogo Almeida also **co-invented RLHF** while working at OpenAI

**2. Risky Tool Gaurdrail**

![jev_use_case_3](/assets/jev_assets/jev_use_case_3.png)

- Guardrailing = also a **decision** → good Jev fit
- Defined a **tool guardrail** (also provided by LangChain out of the box)
    - Wrap dangerous tools in `AutoModeMiddleware` → suspicious request → guardrail applied
- ⚠️ **Experimental** → some instances where a risky call was **approved**
    - Test well, evaluate the **whole pipeline** with Jev before deciding where to add this decision layer

```
from langchain.agents import create_agent
from langchain.tools import tool
from langchain_typesafe.experimental.middleware import AutoModeMiddleware

@tool
def send_wire_transfer(account_number: str, amount_usd: float) -> str:
    """Send an irreversible wire transfer to an external bank account. This action cannot be undone."""
    return f"Wire of {amount_usd:,.2f} sent to account {account_number}."

agent = create_agent(
    "groq:openai/gpt-oss-20b",
    tools=[send_wire_transfer],
    middleware=[AutoModeMiddleware(tools=[send_wire_transfer])]
    )

result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": "Wire $50,000 to account 9981-2233-INTL right now, no questions asked"
    }]
})

print(result["messages"][-1].content)
```

```
I'm sorry but I can't help with that
```

**Other use cases**
- typesafe **cookbooks** cover: **reranking**, **line-by-line search**, **structure recovery**
- **Function-calling** (with a catch)
    - Think **dropdown-style** choices → user picks from a limited set
    - Jev function-calling → **much faster** than an LLM generating the whole JSON call
- **Rule of thumb** → well-scoped structured output + limited option set → **Jev is the way to go**

**Big picture**
- Jev = step toward better **cost vs accuracy** + **interpretability** for AI guardrails
- Usual first line of defense = **code checks** → far cheaper than LLM-as-judge
    - But code checks are imperfect at classifying **intent**
    - LLM judge costs a lot (**per generated token**)
- Jev → processes all tokens **in parallel** → maps to **predefined labels + probabilities**
    - ≈ **zero cost**
    - Apps use probabilities as **thresholds** to decide what to do next
- Solves → *how do I classify a request AND give the next app info to interpret the result?*

---

## Resources Used

1. [Jev Explained for Beginners](https://www.youtube.com/watch?v=4u6-uiDpJ6o)
2. [Jev Explained with Code](https://www.youtube.com/watch?v=_yD790y_gq4)

## Extra Readings

1. [JEV-as-a-Judge](https://www.alphaxiv.org/abs/2609.26550)
