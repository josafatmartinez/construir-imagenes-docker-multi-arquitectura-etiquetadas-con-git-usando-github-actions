# Build Git Tagged Multi-Arch Docker Images with GitHub Actions

Automate building and pushing Docker images that support both AMD64 and ARM64 CPU architectures.

---

By Nick Janetakis

5 min. read

View original

---

Updated on February 20th, 2024 in [#docker](https://nickjanetakis.com/blog/tag/docker-tips-tricks-and-tutorials)

![blog/cards/build-git-tagged-multi-arch-docker-images-with-github-actions.jpg](https://nickjanetakis.com/assets/blog/cards/build-git-tagged-multi-arch-docker-images-with-github-actions-e90531776303dbdd213977768c22580245a573c90edd3e970d91ebc57c259472.jpg)## Automate building and pushing Docker images that support both AMD64 and ARM64 CPU architectures.

**Quick Jump:**[Skimming a Few Set Up Steps](https://nickjanetakis.com/blog/build-git-tagged-multi-arch-docker-images-with-github-actions?ref=dailydev#skimming-a-few-set-up-steps) | [Covering the GitHub Actions Config](https://nickjanetakis.com/blog/build-git-tagged-multi-arch-docker-images-with-github-actions?ref=dailydev#covering-the-github-actions-config) | [Testing It Out](https://nickjanetakis.com/blog/build-git-tagged-multi-arch-docker-images-with-github-actions?ref=dailydev#testing-it-out) | [Demo Video](https://nickjanetakis.com/blog/build-git-tagged-multi-arch-docker-images-with-github-actions?ref=dailydev#demo-video)

**Prefer video? Here it is [on YouTube](https://nickjanetakis.com/blog/build-git-tagged-multi-arch-docker-images-with-github-actions?ref=dailydev#demo-video).**

If you’re not sure what Docker’s Buildx is or want to cover the basics on building multi-CPU architecture Docker images please [check out this post and video](https://nickjanetakis.com/blog/build-multi-cpu-architecture-docker-images-with-buildx).

This post will focus on the GitHub Actions and Docker Hub specifics. We’ll be using the free tier of both services.

If you plan to follow along, here’s a few things you’ll want to set up beforehand:

* Create a [GitHub account](https://github.com/) *(free plan is ok)*
* Create a [Docker Hub account](https://hub.docker.com/) *(free plan is ok)*
* Create a public or private GitHub repo
* Create a public or private Docker Hub repo
* Have a Docker image that you want to publish when git tags are pushed to that repo

We’re going to skim through some of the above steps if you’re not sure how to do them.

## Skimming a Few Set Up Steps

Since you’re reading this post I’m guessing you already have a GitHub account and at least one repo created to work with.

I’ll be using [https://github.com/nickjj/webserver](https://github.com/nickjj/webserver) as a working example for this post.

As for the Docker Hub, after you create an account you can [create a new repository](https://hub.docker.com/repository/create). This is where your Docker images will live for that specific project. You’ll want to pop in a repo name and choose whether or not it’s public or private. Either option works for this case.

Lastly, if you don’t have a Docker image to build and use you can use this very basic hello world `Dockerfile`:

Now you can run this locally `docker image build . -t nickjj/hello-world` which will build and tag that image for your Docker Hub account. Replace `nickjj` with your Docker Hub account.

### Covering the GitHub Actions Config

Create this file in your git repo in `.github/workflows/docker-publish.yml`, feel free to remove the comments. I’ve annotated the file with extra details for this post.

*Here’s an [example of this file](https://github.com/nickjj/webserver/blob/5becee87d67454245e668f5c7e14b2d101883731/.github/workflows/docker-publish.yml) in my webserver project without the comments. Here’s a screenshot of one of the builds, notice the names that appear in the UI based on the config below.*

![blog/github-actions-docker-push.jpg](https://nickjanetakis.com/assets/blog/github-actions-docker-push-371b8e2701bf62c6d05398f6823e3ed19299720f6367bdd081ecbbc8e6796ad4.jpg)

```yaml
# This can be named anything, it'll show up in the GitHub Actions UI.
name: "Dockerpublish"

# This workflow will only run when git tags are pushed that start with `v*`,
# meaning `v123` or `v1.0.0` will work but `0.1.0` won't, if you don't want to
# prefix tags with `v` then remove the `v`.
on:
  push:
    tags:
      - "v*"

jobs:
  # This can also be named anything, it'll show up in the GitHub Actions UI.
  build-and-push-image:
    # Let's use the latest Ubuntu version for our CI environment.
    runs-on: "ubuntu-latest"

    # Most of the steps are things we covered in the Buildx basics post.
    # They are just wrapped up into GitHub Actions instead of being raw shell
    # commands we run manually.
    steps:
      # Literally git checkout the code, this is standard in most actions.
      - name: "Checkout"
        uses: "actions/checkout@v4"

      # Allow the CI environment to emulate multiple CPU architectures.
      - name: "SetupQEMU"
        uses: "docker/setup-qemu-action@v3"

      # Create and configure the Buildx context.
      - name: "SetupDockerBuildx"
        uses: "docker/setup-buildx-action@v3"

      # Log into the Docker Hub (more on this in a bit).
      - name: "LogintoDockerHub"
        uses: "docker/login-action@v3"
        with:
          username: "${{secrets.DOCKERHUB_USERNAME}}"
          password: "${{secrets.DOCKERHUB_TOKEN}}"

      # Push images for X platforms to the Docker Hub (more on this in a bit).
      - name: "Buildandpushimagetags"
        uses: "docker/build-push-action@v5"
        with:
          context: "."
          platforms: "linux/amd64,linux/arm64"
          push: true
          tags: |
            "${{ github.repository }}:latest"
            "${{ github.repository }}:${{ github.ref_name }}"
```

As you can see the config file is pretty concise. Most of the individual actions have additional configuration you can optionally supply. If you Google for them you can find their respective git repos. For example here’s the [docker/build-push-action](https://github.com/docker/setup-buildx-action).

Let’s go over the steps related to logging into the Docker Hub and pushing image tags.

#### Login to Docker Hub

Technically this action supports more than just the Docker Hub for registry access. You can view its [documentation](https://github.com/docker/login-action) for details. When you use the Docker Hub it becomes pretty painless to configure since we can use mostly the defaults.

We’ll need to do a few things before this step will work for us:

1. Create a [personal access token](https://hub.docker.com/settings/security) on the Docker Hub
   * This is how GitHub Actions will use your Docker Hub account
   * This is more secure than using your real Docker Hub password
     * You can revoke the token without needing to re-roll your real password
   * Scope it for `read`, `write` and `delete` if you want to support all of those behaviors
     * In this post we’ll focus on actions that require `write` access
     * If you delete a git tag you could have a separate action `delete` the image tag
2. Create GitHub secrets that are accessible through GitHub Actions
   * This is under your repo Settings -> Security -> Secrets and variables -> Actions
     * [https://github.com/GITHUB_USERNAME/REPO_NAME/settings/secrets/actions](https://github.com/GITHUB_USERNAME/REPO_NAME/settings/secrets/actions)
   * Create a new secret `DOCKERHUB_USERNAME` and use your user name
     * This isn’t really a secret but it’s convenient to have it next to your token
   * Create a new secret `DOCKERHUB_TOKEN` and paste in your personal access token

That’s it! With that configured you can now login to the Docker Hub in a secure way.

#### Build and Push Image Tags

If you’re ok with the config as is then there’s nothing to manually configure here.

It will look for and build a `Dockerfile` in the root of your project and build both AMD64 and ARM64 images then push them to the Docker Hub. This is very similar to the Buildx commands we ran in the previous post.

As for the tags, we’re reaching in and reading [special variables](https://docs.github.com/en/actions/learn-github-actions/variables) that GitHub makes available to use. Referencing `github.repository` in my case will be `nickjj/webserver` as well as `github.ref_name` which when pushing tags will be the exact tag name such as `v0.3.4`.

This avoids us having to manually input the tag name. Yay for automation.

It’s worth pointing out I’m using `github.repository` because my GitHub repo and Docker Hub repo are both named `nickjj/webserver`. You’ll want to ensure that the value matches your Docker Hub repo. If it’s not the same as GitHub then you can use whatever it is for the Docker Hub.

### Testing It Out

After committing your code locally you can run `git tag v1.0.0` and push it all up with `git push origin main --tags`. Our “Docker publish” workflow will start due to the tag matching `v*` and shortly after that everything should get built and pushed.

You can view the status of your workflow under the “Actions” tab in your repo, here’s mine for the webserver project [https://github.com/nickjj/webserver/actions](https://github.com/nickjj/webserver/actions).

If it worked you’ll be able to find your Docker images in your Docker Hub repo, here’s mine for the webserver project [https://hub.docker.com/r/nickjj/webserver/tags](https://hub.docker.com/r/nickjj/webserver/tags).

The video below covers everything in this post.

### Demo Video

#### References

* [https://github.com/docker/compose/releases](https://github.com/docker/compose/releases)
* [https://docs.docker.com/compose/install/linux/#install-the-plugin-manually](https://docs.docker.com/compose/install/linux/#install-the-plugin-manually)

#### Timestamps

* 1:37 – GitHub Action basics
* 2:36 – Only running it when git tags are pushed
* 4:11 – High level overview of the job’s steps
* 5:22 – A few basic preparation steps
* 6:51 – Logging into the Docker Hub with a personal token
* 11:17 – Building and pushing our images
* 14:51 – Checking out the raw commands that were run

**Did you use this set up to push Docker images? Let us know below.**

Like you, I'm super protective of my inbox, so don't worry about getting spammed. You can expect a few emails per month (at most), and you can 1-click unsubscribe at any time. [See what else you&#39;ll get](https://nickjanetakis.com/newsletter) too.
