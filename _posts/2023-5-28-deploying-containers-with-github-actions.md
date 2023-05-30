---
tags:
- Docker
- DevOps
---
The first (official) post of this blog! I'm hoping to post regularly over the summer - I am working full time for IBM Research but have no classes or major competitions so I have time for some projects.

Last semester I encountered Docker/Kubernetes in a couple cyber competitions ([CPTC](https://cp.tc/) and [ISTS](https://ists.io/), respectively) and it inspired me to finally sit down and get better with containerization. I spent a few weeks designing/deploying simple applications to get some experience.  

As someone who knows nothing about DevOps, I started wondering how to automate the deployment process of a containerized application. <b>Herein lies the project.</b>

![]({{ "assets/images/deploying-containers-with-github-actions_ChatGPT.png" | relative_url }})

(Thanks OpenAI. I find myself using ChatGPT mostly in the "I pitch you a problem, you come up with different solutions to it" fashion)

It's too large to include in a screenshot, but it recommended a GitHub Actions workflow that
- Builds the container image
- Pushes it to a registry like Docker Hub
- Connects to the production server via SSH to deploy

A couple tweaks to this plan - I used GitHub Secrets instead of storing credentials in plaintext in the workflow, and I self-hosted a container registry ([Harbor](https://goharbor.io/)) as well as the Github Actions runner itself for an extra challenge.

So my project is:
- Design a Docker application
- Set up infrastructure
  - Container registry (Harbor)
  - GitHub Actions runner
  - "Production" server to run the application
- Set up the workflow
  - Store credentials as GitHub Actions Secrets
  - Write the workflow

<h2>The Application</h2>
Since this is just a PoC, my "web app" is a simple static site.
It consists of two files - a `Dockerfile` and `myPage.html` (the homepage).
The Dockerfile is very straightforward, it starts with the official Apache image from [Docker Hub](https://hub.docker.com/_/httpd), copies my HTML file into the web root, and exposes port 80 to the outside world:
```dockerfile
FROM httpd
COPY ./myPage.html /usr/local/apache2/htdocs/index.html
EXPOSE 80
```
The homepage just contains a header:
```html
<h1>Welcome to my web app!</h1>
```
I initialized this as a git repository and uploaded it to GitHub.

<h2>The Infrastructure</h2>
It's time to set up the self-hosted runner, container registry, and "production environment."

I opted to host all three on the same server, an Ubuntu EC2 instance on AWS. Having the runner on the same server that is deploying the app makes things easy - the workflow doesn't need to connect to a different server via SSH, it can redeploy the app via local commands.

The Github Actions runner setup was as easy as running a few provided commands (under repository Settings -> Actions  Runners) to download/extract the runner and configure it with a token unique to the repository.

Here you can see the runner connecting to GitHub:
![]({{ "assets/images/deploying-containers-with-github-actions_AWS-Runner.png" | relative_url }})

Now on GitHub's side, the runner has been added to the list and shows as idle:
![]({{ "assets/images/deploying-containers-with-github-actions_GitHub-Runner.png" | relative_url }})

[Harbor setup](https://goharbor.io/docs/2.0.0/install-config/) was a little more challenging and involves self-signing certificates for HTTPS. But hey, it comes with a nice web UI (seen below).

![]({{ "assets/images/deploying-containers-with-github-actions_Harbor.png" | relative_url }})

Finally, the only setup needed to be able to deploy the app is installing Docker (good ol' `sudo apt install docker.io`, which I already had to do because Harbor itself is containerized).

<h2>The Workflow</h2>
It's time to actually write the GitHub Actions workflow! First I stored everything needed to access the Harbor instance in Settings -> Secrets and variables -> Actions. Secrets are encrypted at rest and are masked with asterisks in logs, and variables are not. {% raw %}They can be referenced in the workflow with `${{ secrets.SECRET_NAME }}` and `${{ vars.VAR_NAME }}`, respectively.{% endraw %}

![]({{ "assets/images/deploying-containers-with-github-actions_GitHub-Secrets.png" | relative_url }})
![]({{ "assets/images/deploying-containers-with-github-actions_GitHub-Variables.png" | relative_url }})

{% raw  %}
Now for the actual workflow! GitHub looks for anything in the format `.github/workflows/*.yaml` relative to the root of the repository. We only have a single job - steps within a job run in series, different jobs run in parallel.
```yaml
name: deploy-container
on: [push]
jobs:
  deploy-container:
    runs-on: self-hosted
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3

      - name: Build image
        run: docker build . -t ${{ vars.HARBOUR_SERVER }}/myproject/github-actions-test

      - name: Push to registry
        run: |
          echo ${{ secrets.HARBOUR_PASSWORD }} | docker login ${{ vars.HARBOUR_SERVER }} --username ${{ secrets.HARBOUR_USERNAME }} --password-stdin
          docker push ${{ vars.HARBOUR_SERVER }}/myproject/github-actions-test
          docker image rm ${{ vars.HARBOUR_SERVER }}/myproject/github-actions-test

      - name: Redeploy
        run: |
          docker ps --filter label=prod -aq | xargs docker stop | xargs docker rm || true
          docker run -d -p8080:80 -l prod ${{ vars.HARBOUR_SERVER }}/myproject/github-actions-test
```
This is pretty self-explanatory for the most part.
The `on: [push]` means that the job is triggered everytime someone pushes to the repository.
We use a pre-built step that makes a local copy of the repository.
Next we build the image, upload to the registry, and redeploy it - with variable substitution for the credentials and server IP.

The part that gave me the most trouble was definitely the redeploy step.
At one point, I was accidentally stopping containers that were part of Harbor, so the registry would go down.
I found the most elegant solution was to use labels (`-l prod`) when running the containers. It's easy to filter the list of running containers by label, and only stop / remove those.

{% endraw  %}

![]({{ "assets/images/deploying-containers-with-github-actions_GitHub-Runner_Finished.gif" | relative_url }})

And here you can see the final product in action! On the right, I append some text to the web page and push to GitHub. In the bottom left, you can see the self-hosted running picking up the job. In the top left, you can see the changes going live automatically.

