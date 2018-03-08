---
layout: post
title: Chrome in a Docker container
---

There are a number of reasons why one might want to run chrome in a docker
container. For me the principal reason is to allow end-to-end functional 
testing with [TestCafe](https://github.com/DevExpress/testcafe). For my tests 
I use the `chrome:headless` option with `testcafe` and run the tests in a 
docker container. However, for debugging failing tests, I like to run Chrome 
in the same docker container with its display set to my workstation.

To build the docker chrome image, you can use the following `Dockerfile`. 
You might be able to use a leaner image such as `debian`, although I base mine 
off `ubuntu`.

```
FROM ubuntu
RUN apt-get update && \
apt-get install -y wget \
     libxss1 libappindicator1 libindicator7 \
     gconf-service libasound2 libgconf-2-4 libnspr4 \
     fonts-liberation libnss3 lsb-release xdg-utils && \
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb && \
dpkg -i google-chrome*.deb
```

We then build and run the image in a container where we bind mount the X server socket:

```
$ docker build -t chrome .
$ docker run --rm -d -e DISPLAY=:0 -v /tmp/.X11-unix:/tmp/.X11-unix chrome google-chrome --no-sandbox --disable-gpu
```

For Mac, you will need to ensure `XQuartz` is installed. I also needed to disable 
access control with `xhost` and use `--cap-add SYS_ADMIN` when spawning the docker 
container. I found [this article](https://fredrikaverpil.github.io/2016/07/31/docker-for-mac-and-gui-applications/) helpful.
