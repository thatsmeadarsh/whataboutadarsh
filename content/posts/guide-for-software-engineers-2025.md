+++
title = 'How to be Software Engineer in 2025'
date = 2025-08-01T10:00:00+01:00
draft = false
+++
I got an opportunity to do an internal presentation within my organisation. The topic I talked about was "How to be a Software Engineer in 2025?" Let me try to write this as a blog.

## The Evolution of Technology: A Journey Through Time

Many of us have spent years in this fast-paced industry, witnessing firsthand the remarkable evolution of technology. We saw the emergence of Angular and React, transforming traditional web development beyond HTML, JavaScript, and CSS. We experienced how JSTL modernized JSPs and how ESB tools like IIB and Camel revolutionized enterprise integration. We watched as organizations shifted from legacy mainframes to agile, cloud-native systems.

Remember the transition from SOAP to REST, and from XML to sleek, lightweight JSON? We witnessed how Google AMP offered a cost-effective CDN alternative, saw its meteoric rise, and then its ultimate fall. We've adapted as mobile development moved from native Android and Swift to hybrid frameworks, and now we grapple with shifts as cross-platform solutions like Flutter face challenges in the wake of Apple's innovations like Glass UI.

We've adapted to the rise of DevOps, DevSecOps, and the growing footprint of cloud and containerization. The software world embraced microservices, serverless architectures, and resilient cloud platforms, leaving behind the monolithic stacks of yesterday. GitOps, Infrastructure as Code, zero-trust security—the list keeps growing!

And let's not forget the AI revolution—tools like GPT, Copilot, and no-code/low-code platforms are fundamentally reshaping how we build and automate.

Through every technological leap—whether it's the move to mobile-first, the adoption of GraphQL, containerization with Docker and Kubernetes, or the excitement around Edge computing—we have learned, adapted, and thrived. We're here because we never stopped learning, never stopped growing.

So, kudos to all of us for surviving and thriving through these waves of change. Every shift we've seen has only made us more resilient and future-ready!

We have to accept the fact that all these transitions took months, if not years, to be adopted within a large traditional organisation, especially if you are in a regulated industry. The wave of AI transformation will be different because we are seeing tremendous acceptance of MCPs across the industry, though as a protocol it's only less than a year old.

## Essential Skills for Modern Software Engineers

Modern software engineers need certain skills to thrive in this new landscape. These include:

- Changing their attitude from code to collaboration
- Realizing DevOps & DevSecOps will continue to dominate
- Embracing the Gen AI revolution
- Accepting that they have roles beyond just coding
- Understanding the demand for new skills
- Rising as an orchestrator: no one expects you to just do manual coding anymore—you need to become the orchestrator guiding and validating AI systems to complete your tasks
- Recognizing the future is agile, automated, and always about learning

## The Current State of AI in Development

Today, if you show someone how ChatGPT is able to give you answers, they won't be very excited. The same goes if you showcase the possibilities of GitHub Copilot or Amazon Q within an IDE—it's more developer-focused, but still, it doesn't surprise people anymore. Even if you showcase the potential of agentic coding, they aren't amazed. The scenario would have been different if you had shown this some years back, because developers were still working on traditional coding paradigms and had not seen such rapid evolution.

But we were missing something: a proper enterprise-ready solution. In enterprise environments, we don't have the requirement to generate random code to do random tasks. Our need is to develop reliable, scalable, and sustainable solutions. So, it is clear that an enterprise AI solution requires much more capability than just basic prompting or vibe coding.

That's when Anthropic introduced a protocol called MCP (Model Context Protocol). In the industry, we already have hundreds of services that can be consumed via REST calls or some other interface. When a user prompts to validate a pincode, we don't need the AI client to generate code snippets or hallucinate. We want the client to talk to our REST API, comprehend the data as only AI can, and then transform the result into natural language that the user understands. That's where MCP helps. I believe MCPs are the stepping stone to robust, enterprise-ready AI solutions.

### Practical MCP Implementation

I've tried to show some of our developers how I used Playwright MCP with Claude, along with a properly written prompt doc file. It was able to open a browser, and—in our case—it was AEM CMS. It could navigate to a specific path, create a page by choosing the right template, open the page, add predefined components, and configure some sample content. I achieved this by applying proper prompt engineering to leverage Claude's capabilities with MCP support. Effectively, these were some of our Jira tickets for our authors that we managed to automate end-to-end. Some developers were excited, others were less impressed because they had already implemented multiple MCP servers to achieve even more complex tasks. I said, "Great job!" Then I told them Engineers who have never tried MCPs—don't worry. And engineers who have experimented with MCPs—awesome, but don't worry, both of you are outdated (don't take it literally, it was just a joke).

We say consider Gen AI your teammate, and you have to work alongside it as peers. In every project squad, we have real human peers—and none of them is a jack of all trades. All your peer engineers are good at something, and they do it very well. It's time we adapt the same practice to your Gen AI teammate.

## The Game Changer: Anthropic Subagents

Last week, Anthropic launched subagents, designed to work alongside the main AI model, allowing for more specialized and efficient task handling. MCPs were launched as a protocol, and it took some time for technical companies to fully understand and implement them into their workflows. Unlike MCPs, subagents are a feature that can be more easily integrated into existing systems, providing a seamless experience for developers. I truly believe this is going to be a game changer in the enterprise AI world. Imagine the possibilities this will bring, and how agent-to-agent communication will evolve. Note that this isn't direct agent-to-agent communication—it always goes through an orchestration (parent) agent.

So it's extremely important to read their documentation and practice proper prompt engineering to make subagents work the way you want. (https://docs.anthropic.com/en/docs/claude-code/sub-agents) You also need to check their latest sample prompts released on GitHub. This gives you the flexibility to set guardrails and boundaries for every subagent, so your agent will also now adhere to DevSecOps practices. Personally, I would like to see another option in the future where the model can be specified, because not every task or subagent needs to use an expensive model. Sometimes you don't need RAG to fulfill the task—a CAG would be fine. Maybe they will bring this in the future.

So I started off experimenting with subagents and MCPs to see how they can work together to improve our workflows. The architecture looks like this:

{{< figure src="/images/subagents-mcps-architecture.png" alt="Subagents and MCPs Architecture" caption="Integration architecture of subagents and MCPs for workflow automation" width="100%" link="/images/subagents-mcps-architecture.png" target="_blank" >}}

### RTE Plugin Development: A Case Study

I picked a custom RTE plugin for AEM as the development task, and this is how it is automated.

#### Initial 38-Step Automated Process

RTE Plugin Development Workflow Summary: 38-Step Automated Process

**Initiation (Steps 1-9):**
User triggers workflow → System fetches Jira ticket → User confirms to proceed

**Documentation (Steps 10-15):**
System retrieves component documentation from Confluence wiki

**Development (Steps 16-25):**
RTE Developer Agent analyzes codebase → Implements plugin code → User reviews and approves changes

**Version Control (Steps 26-31):**
GitHub Agent commits approved code to repository

**Completion (Steps 32-38):**
System updates Jira ticket status → Notifies user of successful completion

### The Cost Reality Check

Everything looks fancy now! Except the token usage and the cost incurred after one session. If the client bills you $1000 for 8 hours and with the help of AI you complete it in less time—perhaps saving $200—but your Claude usage might have cost you $300. So in reality, you are at a loss of $100. This is not a sustainable model. Once the initial excitement and novelty wear off, teams will need to carefully evaluate the cost–benefit ratio of using AI tools in their workflows.

So I thought about this and came up with a simpler automation solution. I removed certain things that, in my opinion, don't need to be automated or can be achieved in a different way. Here's how I reworked the same task:

{{< figure src="/images/new-subagents-mcps-architecture.png" alt="Subagents and MCPs Architecture" caption="Integration architecture of subagents and MCPs for workflow automation" width="100%" link="/images/new-subagents-mcps-architecture.png" target="_blank" >}}

I removed some automation with smarter approaches and applied possible ways to reduce token usage.

#### Streamlined Agent-Based Process

RTE Plugin Orchestration Workflow Summary: Streamlined Agent-Based Process

**Task Selection:**
User initiates → Task Progress Manager picks the next task from the xx-rte-plugin-feature-tasks file (In Progress or Not Started)

**Analysis:**
Wiki Doc Analyser processes pre-downloaded documentation to extract implementation insights

**Development:**
RTE Component Developer Agent implements code changes with user confirmation for each modification

**Version Control:**
Git Commit Specialist Agent handles code commits and repository management

**Progress Tracking:**
Task Progress Manager updates task status in the tracking file

**Optional Jira Update:**
User can choose to update the Jira ticket with status, comments, and time spent

My experiment shows that by streamlining the workflow and reducing unnecessary automation, we can achieve similar outcomes with lower token usage and costs.

## The Future Role of Software Architects

So, if all these cool jobs are done by software engineers, what is left for architects? We can design these agents. There can be global agents that can be used across different projects and teams, promoting reusability and consistency in our automation efforts. There can be subagents specific to your project needs, allowing for tailored solutions while still leveraging the core capabilities of the global agents. We need to set up the guardrails and only allow privilege where it's required.

## Conclusion

I am too excited for the future of AI-driven development!

At the end, we never stopped learning, never stopped growing.
