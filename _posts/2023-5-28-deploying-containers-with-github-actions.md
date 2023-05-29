---
tags:
- Docker
- DevOps
---
The first (official) post of this blog! I'm hoping to post regularly over the summer - I am working full time for IBM Research but have no classes or major competitions so I have time for some projects.

Last semester I encountered Docker/Kubernetes in a couple cyber competitions ([CPTC](https://cp.tc/) and [ISTS](https://ists.io/), respectively) and it inspired me to finally sit down and get better with containerization. I spent a few weeks designing/deploying simple applications to get some experience.  

I started wondering how to automate the deployment process of a containerized application. <b>Herein lies the project.</b>

![]({{ "assets/images/deploying-containers-with-github-actions_ChatGPT.png" | relative_url }})

(Thanks OpenAI. I find myself using ChatGPT mostly in the "I pitch you a problem, you come up with different solutions to it" fashion)

It's too large to include in a screenshot, but it recommended a GitHub Actions workflow that
- builds the container
- pushes it to a registry like Docker Hub
- connects to the production server via SSH to deploy

A couple tweaks to this plan - I used GitHub Secrets instead of storing credentials in plaintext in the workflow, and I self-hosted a container registry ([Harbor](https://goharbor.io/)) for an extra challenge.

WIP - I'll finish this post tomorrow. The workflow is working but needs a little polishing.
