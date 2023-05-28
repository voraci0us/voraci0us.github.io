This summer, I'm working for IBM Research based out of Yorktown Heights, NY. It's been a great experience so far and I'll probably write about it closer to the end of the summer. But for now... no part-time jobs, competitions, or classes. In other words, <b>I have spare time</b>. Let's fix that.

I've gradually been dipping my toes into DevOps by getting familiar with Terraform/Ansible/Docker/Kubernetes. It feels like a good time to tie it together by building a full CI/CD pipeline to deploy an application.

I proposed a project to ChatGPT:
"I have a Gihub project that has a Dockerfile. I have a production server running the Docker image that the project creates. Everytime a push is made to the Github project, I want to rebuild the Docker image and redeploy it."

I find myself using ChatGPT mostly in the "I pitch you a problem, you come up with different solutions to it" fashion. In this case, it came back recommending a GitHub Actions workflow that builds the container, pushes it to a registry like Docker Hub, and SSHes into the production server to deploy. I said I didn't want credentials in the workflow in plaintext, and it recommended GitHub Secrets.

I have not used GitHub Actions (or Secrets) before. This sounds like a fun challenge. To make it a little trickier, I'm hosting my own GitHub Actions runner and container registry (Docker Harbour, a CNCF project).

In the works currently, I'll update this post when it's done.