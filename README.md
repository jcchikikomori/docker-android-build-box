# Docker Android Build Box

![docker icon](https://dockeri.co/image/jccdevbox/docker-android-build-box/)
[![Build Status](https://travis-ci.com/jcchikikomori/docker-android-build-box.svg?branch=develop)](https://travis-ci.com/jcchikikomori/docker-android-build-box)


## Introduction

An optimized **docker** image includes **Android**, **Kotlin**, **Flutter sdk**.
Based on [Ming Chen's Android Build Box](https://github.com/mingchen/docker-android-build-box)

## What Is Inside

It includes the following components:

* Ubuntu 18.04
* Android SDKs
  * 25
  * 26
  * 27
  * 28
  * 29
  * 30
* Android build tools:
  * 25.0.0 25.0.1 25.0.2 25.0.3
  * 26.0.0 26.0.1 26.0.2
  * 27.0.1 27.0.2 27.0.3
  * 28.0.1 28.0.2 28.0.3
  * 29.0.2 29.0.3
  * 30.0.0
* Android NDK r21
* Android Emulator
* TestNG
* Python 2, Python 3
* Node.js, npm, React Native
* Ruby, RubyGems
* fastlane
* Kotlin 1.3
* Flutter 1.17.1

## Difference between the original?

* Upgraded to Ruby 2.7 from a PPA repository
  (`ppa:brightbox/ruby-ng`)
* Fastlane
* Android SDK legacy support:
  * 24
  * 23
  * 22
  * 21

## Pull Docker Image

The docker image is publicly automated build on [Docker Hub](https://hub.docker.com/r/jccdevbox/docker-android-build-box/)
based on the Dockerfile in this repo, so there is no hidden stuff in it. To pull the latest docker image:

```sh
docker pull jccdevbox/docker-android-build-box:latest
```

## Usage

### Use the image to build an Android project

You can use this docker image to build your Android project with a single docker command:

```sh
cd <android project directory>  # change working directory to your project root directory.
docker run --rm -v `pwd`:/project jccdevbox/docker-android-build-box bash -c 'cd /project; ./gradlew build'
```

Run docker image with interactive bash shell:

```sh
docker run -v `pwd`:/project -it jccdevbox/docker-android-build-box bash
```

### Build an Android project with [Bitbucket Pipelines](https://bitbucket.org/product/features/pipelines)

If you have an Android project in a Bitbucket repository and want to use the pipeline feature to build it,
you can simply specify this docker image.
Here is an example of `bitbucket-pipelines.yml`:

```yml
image: jccdevbox/docker-android-build-box:latest

pipelines:
  default:
    - step:
        caches:
          - gradle
          - gradle-wrapper
          - android-emulator
        script:
          - bash ./gradlew assemble
definitions:
  caches:
    gradle-wrapper: ~/.gradle/wrapper
    android-emulator: $ANDROID_HOME/system-images/android-21
```

#### Fastlane + Firebase App Distribution

Here's the example for deploying your app into Firebase's App Distribution

```yml
image: jccdevbox/docker-android-build-box:latest

pipelines:
  default:
    - step:
        name: "Fastlane to Firebase"
        caches:
          - bundler
          - gradle
          - gradle-wrapper
        script:
          - echo "$KEYSTORE_PROPERTIES" | base64 -d > keystore.properties
          - echo "$KEY_BASE64" | base64 -d > app/keystore
          - echo "$GOOGLE_SERVICES_API" | base64 -d > app/google-api.json
          - echo "$GOOGLE_SERVICES_KEY" | base64 -d > app/google-services.json
          - echo "$FASTLANE_ENV" | base64 -d > fastlane/.env
          - npm install -g firebase-tools # TODO: apparently there is no firebase CLI
          - export FIREBASE_TOKEN=$FIREBASE_TOKEN # https://firebase.google.com/docs/cli#cli-ci-systems
          - export CI=1 # https://github.com/fastlane/fastlane/issues/13504
          - cp -f Gemfile /tmp/Gemfile
          - cp -f Gemfile.lock /tmp/Gemfile.lock
          - mkdir -p /tmp/fastlane
          - cp -f fastlane/Pluginfile /tmp/fastlane/Pluginfile
          - echo "$FASTLANE_ENV" | base64 -d > /tmp/fastlane/.env
          - bundle install
          - bundle exec fastlane init
          - bundle exec fastlane env
          - bundle exec fastlane install_plugins
          - bundle exec fastlane release
        artifacts:
          - app/build/outputs/apk/staging/release/*.apk
definitions:
  caches:
    gradle-wrapper: ~/.gradle/wrapper
    android-emulator: $ANDROID_HOME/system-images/android-21
```

The caches are used to [store downloaded dependencies](https://confluence.atlassian.com/bitbucket/caching-dependencies-895552876.html) from previous builds, to speed up the next builds.

### Build a Flutter project with [Github Actions](https://github.com/features/actions)

Here is an example `.github/workflows/main.yml` to build a Flutter project with this docker image:

```yml
name: CI

on: [push]

jobs:
  build:

    runs-on: ubuntu-18.04
    container: jccdevbox/docker-android-build-box:latest

    steps:
    - uses: actions/checkout@v2
    - uses: actions/cache@v1
      with:
        path: /root/.gradle/caches
        key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle') }}
        restore-keys: |
          ${{ runner.os }}-gradle-
    - name: Build
      run: |
        echo "Work dir: $(pwd)"
        echo "User: $(whoami)"
        flutter --version
        flutter analyze
        flutter build apk
    - name: Archive apk
      uses: actions/upload-artifact@v1
      with:
        name: apk
        path: build/app/outputs/apk
    - name: Test
      run: flutter test
    - name: Clean build to avoid action/cache error
      run: rm -fr build
```

### Run an Android emulator in the Docker build machine

Using guidelines from...

* https://medium.com/@AndreSand/android-emulator-on-docker-container-f20c49b129ef
* https://spin.atomicobject.com/2016/03/10/android-test-script/
* https://paulemtz.blogspot.com/2013/05/android-testing-in-headless-emulator.html

...You can write a script to create and launch an ARM emulator, which can be used for running integration tests or instrumentation tests or unit tests:

```sh
#!/bin/bash

# Download an ARM system image to create an ARM emulator.
sdkmanager "system-images;android-16;default;armeabi-v7a"

# Create an ARM AVD emulator, with a 100 MB SD card storage space. Echo "no"
# because it will ask if you want to use a custom hardware profile, and you don't.
# https://medium.com/@AndreSand/android-emulator-on-docker-container-f20c49b129ef
echo "no" | avdmanager create avd \
    -n Android_4.1_API_16 \
    -k "system-images;android-16;default;armeabi-v7a" \
    -c 100M \
    --force

# Launch the emulator in the background
$ANDROID_HOME/emulator/emulator -avd Android_4.1_API_16 -no-skin -no-audio -no-window -no-boot-anim -gpu off &

# Note: You will have to add a suitable time delay, to wait for the emulator to launch.
```

Note that x86_64 emulators are not currently supported. See [Issue #18](https://github.com/mingchen/docker-android-build-box/issues/18) for details.

## Docker Build Image

If you want to build the docker image by yourself, you can use following command.
The image itself is around 5 GB, so check your free disk space before building it.

```sh
docker build -t docker-android-build-box .
```

## Contribution

If you want to enhance this docker image or fix something,
feel free to send [pull request](https://github.com/mingchen/docker-android-build-box/pull/new/master).


## References

* [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)
* [Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)
* [Build your own image](https://docs.docker.com/engine/getstarted/step_four/)
* [uber android build environment](https://hub.docker.com/r/uber/android-build-environment/)
* [Refactoring a Dockerfile for image size](https://blog.replicated.com/refactoring-a-dockerfile-for-image-size/)
* [Label Schema](http://label-schema.org/)
