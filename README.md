# LLM Evaluation Harness

An offline evaluation harness for an LLM application that reads customer support email
threads and turns them into structured support tickets. The harness measures what a model
change or a prompt change actually does to accuracy, latency, and cost.

I built it to learn how eval harnesses work. The harness is the project; the ticket
extraction LLM application is just the thing it measures.

## At a glance

- **What models I compared:** Claude Haiku 4.5 vs GPT-5.4 mini, across 50 test cases
- **What I found:** prompt changes did more for improving accuracy than switching models 
  did. The two models performed nearly identically on accuracy and latency; however, 
  GPT-5.4 mini was 38% cheaper on this workload
- **How the LLM outputs were scored:** exact match for the fields with one right answer, 
  and a Google Gemini judge for the free-text fields. The judge was checked against 
  labels from two independent human annotators before its verdicts were allowed to count

## What I learned

### You can't unit test a model's output

I've spent my career building applications where the same input always produces the same 
output. That makes testing straightforward, you assert that a function returns the value 
you expect, and if it doesn't, the build fails. LLMs don't work that way. Send the same 
input through twice and you can get two different outputs back, and both of them could be 
correct. There's no single output to assert against, so there's nothing for a unit test 
to check.

An eval harness is the solution. Instead of checking whether one input returned the
expected output, it runs a whole set of cases and scores how good the results are across
all of them. That gives you a number you can compare against a previous number, which is
the thing a regular test suite can't give you.

### LLM judges exist because hand labeling doesn't scale

The annotation step in an evaluation is crucial. It gives you the opportunity to review 
the application's output, decide if the free-text output passes or fails, and explain why. 
This steps is what gives you the feedback you need to judge and improve the application. 
However, this step is very time consuming and doesn't scale. Thankfully, we can create LLM 
judges to work alongside human judges to help with the volume.

### An LLM judge can only be trusted if you validate it

An LLM judge can only be trusted if you've verified its output against a human labeled 
calibration set.

I validated my LLM judge against 140 outputs that a second annotator and I 
labeled by hand, independently, without seeing each other's answers. We disagreed on 13 
of them, and I made the final call on each one and recorded it. That gave me two things: 
a set of answers I trust to check the judge against, and a measure of how often two 
humans doing the same job even agree with each other, which turns out to be the realistic 
ceiling for what you can expect from a judge.

### Golden sets test models, calibration sets test judges

The 50 golden cases measure the models. Each one is a support email plus the ticket I
expect back, so a model's score is how close it got. The emails are synthetic, generated
with Gemini and Claude, then curated by hand.

The 35 calibration emails meausure the judge. These emails are also sythetic, generated with 
Cluade, but then I planted specific defects: dropping the one thing the customer
actually asked for, adding a detail the email never mentioned, swapping one person's name
for another, etc. That brought the failure rate to about 23%, which givs the judge an opportunity 
to demonstrate it can tell good from bad.

### The model mattered less than the application

I tested Claude Haiku 4.5 against GPT-5.4 mini across all 50 golden test cases, and they tied on
accuracy and latency.

| | Haiku 4.5 | GPT-5.4 mini |
|---|---|---|
| Composite score, all 50 items | **93.24** | **94.57** |
| Straightforward emails (32) | 93.30 | 95.83 |
| Adversarial emails (18) | 93.12 | 92.33 |
| Latency (median) | 1.5s | 1.5s |

What I did see was a trade-off: Haiku got the literal fields right every single time (customer name, 
order ID, product name), and GPT-5.4 mini wrote better summaries (94% vs 79%) and read urgency 
better (91% vs 84%).

The only place they genuinely separated was price: across the same 150 calls, GPT-5.4 
mini's model spend was $0.20 against Haiku's $0.33, about 38% less.

However, the more useful thing I learned happened while I was building. As I adjusted the prompts, 
both models got better, and those gains were bigger than the gap between the two models 
ever was. I was surprised how much of the quality was sitting in the application rather 
than in the model, and the best part is that the harness let me watch it happen instead of guessing.

### Models favor their own family

While I was deciding which model should be the judge, I read that models tend to favor
their own family, an OpenAI judge goes easy on OpenAI output. My candidates were
Anthropic and OpenAI, so I picked a judge from Google. I used the same logic on the data:
most of the golden emails came from a model outside both candidates' families, and the
calibration emails came from outside the judge's.

However, a judge with no stake in either model can still be tougher on one of them, and
that's the version that wrecks an A-vs-B comparison. So I checked the judge's agreement
with my labels separately for each candidate, and it agreed with me a lot more on GPT-5.4
mini than on Haiku. Turns out it likes longer answers, and Haiku writes short ones.

## How I built this

I used Claude Code and followed spec-driven development. Before any code, there was a
constitution, a spec, a decisions file, and a plan broken into 20 tickets. Claude worked
against those documents and I reviewed the results ticket by ticket.

The parts I did by hand: designing the failure cases in the test set, labeling 140
calibration rows, and reviewing the 13 rows where my second annotator and I disagreed.

## What's next

Two things I want to focus on next.

The first is cost. If two frontier models are this interchangeable on a task like this,
the question isn't which one is best, it's how cheap you can go before quality actually
breaks. I want to keep stepping down to cheaper models until accuracy or latency falls
off, and find where that cliff is.

The second is harder problems. Multi-ticket extraction, where one thread produces an
unknown number of tickets, so I'd have to work out how to score variable-length output.