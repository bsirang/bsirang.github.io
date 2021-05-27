---
layout: post
title: "Experimenting with ChibiOS and Bazel with bazel-embedded"
categories: [chibios, bazel, stm32]
---
### Background

#### ChibiOS
I recently started looking into [ChibiOS](https://www.chibios.org/dokuwiki/doku.php) as a viable open-source RTOS for a new STM32F4 microcontroller project. I had used FreeRTOS somewhat extensively on Cortex-M4 devices, and I was curious to try something new. The biggest draw towards ChibiOS for me was the well-defined HAL that's included. Not just that but it ships with an STM32F4 implementation of the HAL. The STM32F4 version appeared to be the primary supported implementation also. At a glance, the OS design, documentation, and APIs all seemed clean. The HAL and the real-time kernel are actually two separate projects entirely ([ChibiOS/HAL](https://www.chibios.org/dokuwiki/doku.php?id=chibios:products:hal:start) and [ChibiOS/RT](https://www.chibios.org/dokuwiki/doku.php?id=chibios:products:rt:start)). I've written "RTOS-aware" microcontroller peripheral drivers plenty of times in the past, and the prospect of skipping that altogether sounded very appealing.

ChibiOS has a matrix of licensing options, including commercial licenses, which I found to be reasonably priced. Including free for low volumes. See more details [here](https://www.chibios.org/dokuwiki/doku.php?id=chibios:licensing:start).

#### Bazel
In parallel, I had been wanting to use [Bazel](https://bazel.build/) as a build system for my firmware projects. Bazel is a powerful build system that prides itself on hermeticity, which is a fancy way of saying that 100% reproducible builds can be guaranteed across developer machines, and over time. While that sounds simple enough, it's something that's been very difficult to guarantee with traditional build systems such as make et. al. It's why many developers get in a habit of running `make clean` often because they don't trust their build system after being bitten a few times. Hermetic builds unlock the power of remote caching, which can yield huge performance gains. Hermetic builds also guarantee that developers are building and testing the exact same artifacts as the official CI builds.

When I first familiarized myself with Bazel, I shied away from the idea of using Bazel for a microcontroller firmware project. This was primarily because I couldn't risk hitting a dead-end after burning a lot of time trying to get something to work given that it isn't Bazel's primary use case. For example, I wasn't sure how easy it would be to integrate the required platform-specific toolchain or how to feed in a custom linker script to the `cc_binary` rule.

The next time around I decided to investigate it again. That's when I came across [bazel-embedded](https://github.com/silvergasp/bazel-embedded). This project provided everything I needed to make something work, so I decided to give it a shot.

### The Experiment

I ended up with a `WORKSPACE` file like so:

~~~ python
workspace(name = "chibios-bazel")

load("@bazel_tools//tools/build_defs/repo:git.bzl", "git_repository")

git_repository(
    name = "bazel_embedded",
    commit = "14d51da3c1de4c7b8b7ce78e87e4f25d9802bee4",
    remote = "https://github.com/silvergasp/bazel-embedded.git",
    shallow_since = "1616471159 +0800"
)

load("@bazel_embedded//:bazel_embedded_deps.bzl", "bazel_embedded_deps")

bazel_embedded_deps()

load("@bazel_embedded//platforms:execution_platforms.bzl", "register_platforms")

register_platforms()

load(
    "@bazel_embedded//toolchains/compilers/gcc_arm_none_eabi:gcc_arm_none_repository.bzl",
    "gcc_arm_none_compiler",
)

gcc_arm_none_compiler()

load("@bazel_embedded//toolchains/gcc_arm_none_eabi:gcc_arm_none_toolchain.bzl", "register_gcc_arm_none_toolchain")

register_gcc_arm_none_toolchain()

load("@bazel_embedded//tools/openocd:openocd_repository.bzl", "openocd_deps")

openocd_deps()
~~~

Next I needed to make an attempt at "Bazel-ifying" ChibiOS. One way to do that is to write a `BUILD` file that lives in my repository and then tell Bazel to fetch a version of ChibiOS to apply the build file to with the `http_archive` rule. I used version `20.3.3`. I ended up adding this to my `WORKSPACE`:

~~~ python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "chibios",
    url = "https://github.com/ChibiOS/ChibiOS/archive/refs/tags/ver20.3.3.tar.gz",
    sha256 = "e6e5cafa5a74346e20ad5fc735d424a073856a0ecb6d32f4a81a477e501c15e7",
    build_file = "@//example:chibios.BUILD",
    strip_prefix = "ChibiOS-ver20.3.3",
)
~~~

The magic here lives in the [chibios.BUILD](https://github.com/bsirang/chibios-bazel/blob/main/example/chibios.BUILD) file. This developed somewhat painstakingly by studying the ChibiOS code structure and then throwing in some trail-and-error on top. This build file specifies all the source files I needed for my project. There's a chance it's missing something you need, so if you get an `undefined reference` linker error for a ChibiOS component chances are you need to tweak the sources in the build file.

Check out the [skeleton project here](https://github.com/bsirang/chibios-bazel).
