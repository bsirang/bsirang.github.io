---
layout: post
title: "Github Workflows With Docker and Bazel Caching"
categories: [bazel, docker, github, workflows, actions]
---

I recently configured [Github Actions](https://github.com/features/actions) for continuous integration on a couple of my projects with good results. One such project is called [trellis](https://github.com/agtonomy/trellis), which I'll discuss in a future post. With Github Actions you can define a workflow for continuous integration, which can run on ephemeral build agents (think something like AWS lambdas).

## Workflows
 The workflow I created is defined in a yaml file that lives in `.github/workflows/main.yaml`, which is picked up by Github automatically.

### Triggers
 Inside the yaml file the triggers are defined. In my case, I want to trigger the workflow on both pull requests and updates to `master`.

 ~~~yaml
 on:
   pull_request:
     branches:
       - master
   push:
     branches:
       - master
~~~

### Jobs
Workflows are composed of jobs. Jobs can run concurrently, but the steps within jobs run sequentially. In my case I have one job that builds all targets, runs all testable targets, and runs my various linters.

~~~yaml
jobs:
  bazel_build_job:
    runs-on: ubuntu-latest
    name: Bazel build everything
    steps:
      - name: Checkout step
        id: checkout
        uses: actions/checkout@v2
...
~~~

### Steps
As mentioned above, jobs are composed of steps. Steps are defined via _actions_. Actions can come from publicly available repositories such as the [checkout action](https://github.com/actions/checkout) above, which comes from the official https://github.com/actions site. You can also define custom actions that live in your repository, which I'll go over later.

## Docker Image Build (With Caching)
For my project I have a `Dockerfile` which defines a build & runtime environment for for my code. I didn't want to necessarily host the Docker image in a repository somewhere, so I decided to see how I could get the image to be built as part of the job. This `Dockerfile` serves to provide a consistent environment in which to run the CI checks against my project.

The following three actions came to the rescue:

~~~yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v1
- name: Cache Docker layers
  uses: actions/cache@v2
  with:
    path: /tmp/.buildx-cache
    key: ${{ runner.os }}-buildx-${{ github.sha }}
    restore-keys: |
      ${{ runner.os }}-buildx-
- name: docker buildx
  uses: docker/build-push-action@v2
  with:
    context: ./docker
    push: false
    load: true
    tags: agtonomy/trellis-runtime:latest
    cache-from: type=local,src=/tmp/.buildx-cache
    cache-to: type=local,dest=/tmp/.buildx-cache-new
~~~

The first action, `docker/setup-buildx-action`, is used to set up [buildx](https://docs.docker.com/buildx/working-with-buildx/), which is a Docker plugin that uses [BuildKit](https://github.com/moby/buildkit).

The next action, `actions/cache`, is used to configure disk volumes that persist from build to build using a keying mechanism. This is what enables our image layers to be cached.

The last action, `docker/build-push-action` allows us to actually build the image using Buildx. We set `push` to false because we don't actually want to push the image to a registry. We set `load` to true, because we do want to export the resultant image for user with Docker. The `context` key tells the action where to find the `Dockerfile` in the repository.

There is an additional step needed at the end to keep the cache from growing indefinitely, and that is to remove the old cache and move the output to the original location:

~~~yaml
- name: Move cache
  run: |
    rm -rf /tmp/.buildx-cache
    mv /tmp/.buildx-cache-new /tmp/.buildx-cache
~~~

Special thanks to this blog post for guidance! https://evilmartians.com/chronicles/build-images-on-github-actions-with-docker-layer-caching

## Custom Actions

With the Docker image built, I now want to develop custom actions to be run within the Docker image.

To do this I create an `action.yml` file for each of my custom actions:
~~~bash
$ find .github/actions/ -iname action.yml
.github/actions/clang-format/action.yml
.github/actions/bazel-build/action.yml
.github/actions/shellcheck/action.yml
.github/actions/buildifier/action.yml
~~~

As the directory names suggest, these actions are for running the Bazel build and running a few linters ([clang-format](https://clang.llvm.org/docs/ClangFormat.html), [shellcheck](https://github.com/koalaman/shellcheck), and [buildifer](https://github.com/bazelbuild/buildtools/blob/master/buildifier/README.md)). All of these tools were placed in the Docker image that was already built earlier.

Here's what the clang-format action looks like:

~~~yaml
name: 'clang-format check'
description: 'C++ source code format linter'
runs:
  using: 'docker'
  image: 'agtonomy/trellis-runtime:latest'
  args:
    - run_clang_format
~~~

It's basically saying -- invoke `docker run` on the image `agtonomy/trellis-runtime:latest` and pass in `run_clang_format` as an argument. `run_clang_format` is actually a bash script that simply invokes clang-format on the source files in the repository. As part of `docker run` this action actually passes a lot of environment variables as well as several volume mounts to the Docker environment. This is how the source code that was checked out becomes visible within the Docker environment.

Here's the full command from an actual build:

~~~bash
/usr/bin/docker run --name agtonomytrellisruntimelatest_9376a9 --label e1cc51 \
--workdir /github/workspace --rm -e HOME -e GITHUB_JOB -e GITHUB_REF \
-e GITHUB_SHA -e GITHUB_REPOSITORY -e GITHUB_REPOSITORY_OWNER -e GITHUB_RUN_ID \
-e GITHUB_RUN_NUMBER -e GITHUB_RETENTION_DAYS -e GITHUB_RUN_ATTEMPT \
-e GITHUB_ACTOR -e GITHUB_WORKFLOW -e GITHUB_HEAD_REF -e GITHUB_BASE_REF \
-e GITHUB_EVENT_NAME -e GITHUB_SERVER_URL -e GITHUB_API_URL \
-e GITHUB_GRAPHQL_URL -e GITHUB_WORKSPACE -e GITHUB_ACTION -e GITHUB_EVENT_PATH \
-e GITHUB_ACTION_REPOSITORY -e GITHUB_ACTION_REF -e GITHUB_PATH -e GITHUB_ENV \
-e RUNNER_OS -e RUNNER_NAME -e RUNNER_TOOL_CACHE -e RUNNER_TEMP \
-e RUNNER_WORKSPACE -e ACTIONS_RUNTIME_URL -e ACTIONS_RUNTIME_TOKEN \
-e ACTIONS_CACHE_URL -e GITHUB_ACTIONS=true -e CI=true \
-v "/var/run/docker.sock":"/var/run/docker.sock" \
-v "/home/runner/work/_temp/_github_home":"/github/home" \
-v "/home/runner/work/_temp/_github_workflow":"/github/workflow" \
-v "/home/runner/work/_temp/_runner_file_commands":"/github/file_commands" \
-v "/home/runner/work/trellis/trellis":"/github/workspace" \
agtonomy/trellis-runtime:latest  "run_clang_format"
~~~

Note the volume mounts. I exploit the workspace volume mount for the Bazel cache, which I'll get into below.

## Bazel Build With Cache (Within Docker Container)
In order to provide a persistent volume mount for the Bazel cache, I had to specify another cache action. This time I used this path for the cache path: `/home/runner/work/trellis/trellis/.cache/bazel`. If you noticed this Docker volume mount above `-v "/home/runner/work/trellis/trellis":"/github/workspace"`, this means inside the Docker environment the Bazel cache will live at `/github/workspace/.cache/bazel`.

Basically, I piggy-backed off of an existing volume mount to expose the cache that lives on the host system from inside the Docker container.

In my custom action for running Bazel, I directed the disk and repository caches to the persistent volume using these flags:

~~~bash
BAZEL_CACHE_DIR="/github/workspace/.cache/bazel"
bazel test --repository_cache="$BAZEL_CACHE_DIR/repository" --disk_cache="$BAZEL_CACHE_DIR/disk" ...
~~~

After doing this I was almost there, but there was one last snag. The post cache Github step, which archives the cache for future runs, was failing on _permission denied_ errors. It dawned on me that this is happening because the Bazel cache, which is being written to from inside the Docker container, is owned by `root`.

To work around this, I added a small hack to change ownership after the `bazel` command above completed.
~~~bash
chown -R 1001:1001 "$BAZEL_CACHE_DIR"
~~~

I call this a hack because the UID/GID of 1001 is hardcoded here. I found this by trial and error knowing that UIDs tend to start from 1000 and are sequential. This is the user ID that Github's `ubuntu-latest` VM is using to invoke the workflow. Ideally these UID/GID would be parameterized so it isn't hardcoded. Given this is not likely to change, I was okay with the slight hack.

### What about Bazel's remote cache feature?
Bazel does support a remote cache natively, which can be hosted on a Google Cloud Storage (GCS) bucket, for example. Even if you plan on using a remote cache, you can get major wins by using Github's cache action to host your Bazel cache. This is because the throughput/latency bottleneck of accessing your remote cache from within Github's cloud infrastructure will take significantly longer.

## Conclusion

With a small set of `yaml` files in your repository, it's possible to have Github run ephemeral workers to run any of your arbitrary CI tasks with cache disk volumes in between runs all with 0% effort in maintaining the cloud resources or attempting to integrate another CI provider.

In my case the cached Docker build takes almost exactly 1 minute to complete, half of which was due to loading the image to Docker after it was built.

The cached Bazel build takes about 24 seconds to complete, all of which was the overhead of extracting the cache and analyzing it.

With Docker and Bazel build caches, you can have CI builds on relatively large codebases that take just a couple minutes. This is a big win!

The source referenced in this post can be found here: https://github.com/agtonomy/trellis/tree/master/.github
