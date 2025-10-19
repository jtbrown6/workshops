# This repo contains different projects that we are planning to build to perform deep dives on technologies with hands on lessons

# Scenarios

Below are different scenarios that we want to create labs for to allow users to deep dive on the different content. You are to leverage the different prompts as inputs first found below which are either Tech Deepdive or Problem Solving. The prompt that should be leveraged is one or the other and each Scenario will determine which to use.

## Codex Scenario Prompt
Create a new directory which consist of writing a step by step workshop that iteratively builds to create the users scenario mentioned. You are to act as an expert in the domains of the scenario and teach it to an intermediate learner as if you were hosting a screen recording and the user is following along with the workshop and code. As you are writing the workshop, connect any related surrounding technologies and what it depends on, walk through how it works under the hood, define concepts in plain language and any common misconceptions all in a conversational tone and any analogies or simplified examples to aid understanding. 

Other style requirements for the workshop:
- build the explanation layer-by-layer
- Include simulated questions you would expect from learners: “You might be wondering why…”

**Scenario 1:** 
- Lets create a workshop on using configmaps and volumes within kubernetes. The webapp should be very simple, like nginx, and iteratively go through the process of teaching users how to leverage configmaps to inject data into the application as a env_var and also as a file. The application should have a simple UI which can prove out the hot-loading nature of using configmaps as volumes to show that after applying a new configmap, the application can see the new values. This should also include secret injection as well. The final lessons would be to convert this scenario into a simple helm chart and intially show the user a base deployment without any volumes and slowly iterate on it and inject the configmaps etc to mimic the same thing that was done but using helm instead. 



# Prompts

## 1. Tech DeepDive

```bash
Create a step-by-step think-aloud explanation of [TECHNOLOGY OR CONCEPT] as if you’re an expert teaching it to an intermediate learner during a screen-recorded session.

Style Requirements:
- Use first-person, conversational tone
- Narrate your thought process: “Let’s break this down…”, “First, I want to understand…”
- Build the explanation layer-by-layer, starting from the core ideas
- Include simulated questions you would expect from learners: “You might be wondering why…”
- Mention what resources or docs you’d consult to verify or clarify things
- Use analogies or simplified examples to aid understanding
- Reference real-world use cases or implementation contexts
- Include “mental Google searches” where relevant: “I’d probably search for ‘how does [X] work internally’”

Structure:
1. Define the concept in plain language
2. Identify real-world relevance or use cases
3. Break down the major components or architecture
4. Walk through how it works under the hood
5. Connect to surrounding technologies (what it depends on or interacts with)
6. Mention any common misconceptions or pitfalls
7. Summarize the key mental model someone should retain
8. Optionally: Add follow-up paths for deeper learning

Target audience: [Curious developers / junior engineers / interview candidates]

Make it feel like listening to an expert think aloud while unpacking a topic from the ground up — curious, thoughtful, and educational, not overly polished.

# With a real example
Create a think-aloud walkthrough of Kubernetes as if you're an experienced cloud engineer teaching it to a junior developer during a mentorship session.

Include:
- First-principles explanation of why Kubernetes exists
- Real-world analogy (e.g., shipping containers, fleet managers)
- Explanation of Pods, Nodes, Services, Deployments
- Simulated learner questions like "How does it know where to run pods?"
- References to `kubectl`, YAML files, and common tools
- Common gotchas or misconceptions
- Summary of key mental models and follow-up resources
```

## 2. Problem Solving

```bash
Create a step-by-step walkthrough guide that simulates an expert solving [PROBLEM] as if thinking aloud during a technical interview.

Style requirements:
- Write in first-person, conversational tone
- Show incremental problem-solving (don't jump to the final solution)
- Include "mental Google searches" - what you would search for at each step
- Explain the reasoning behind each decision ("Why this approach?")
- Reference documentation or resources you would consult
- Build the solution piece by piece, with each step informing the next
- Include both successful steps AND course corrections when needed
- Add commentary on what you're thinking: "Let me think about...", "I need to check..."

Structure:
1. Start by understanding the problem and requirements
2. Examine existing context/infrastructure
3. Research relevant technologies/patterns
4. Plan the solution architecture
5. Implement step-by-step, explaining each piece
6. Include testing/verification steps
7. Summarize key insights and decisions


Target audience: [Someone learning this topic / Interview candidate / etc.] Make it feel like watching someone's screen recording while they narrate their thought process - authentic, educational, and showing the real path to the solution.
```