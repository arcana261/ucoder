title: Building Spring Boot Application inside Docker file
author: Mohamad mehdi Kharatizadeh
tags:
  - Java
  - Docker
  - Spring
  - SpringBoot
  - Heroku
categories: []
date: 2017-12-24 00:56:00
---
A little ago, I was investigating how to properly build my Spring Boot application inside a dockerfile so I could easily deploy it to heroku. I looked up the net and I came across solutions which encouraged to copy built "JAR" file into the docker image. *The Thing I dislike the most!*. Since copying a JAR file directly into a container image is somewhat unclean and dirty! A clean docker image which does the build is also safe to run automated tests on it so we could be sure there is no problem on the version of JDK we are deploying etc. So I came up with the following dockerfile:


```docker
FROM openjdk:8u151-jdk

RUN \
    cd / && \
    git clone https://github.com/arcana261/RahanSazehApiServer.git && \
    cd /RahanSazehApiServer && \
    git checkout -b new-version origin/new-version && \
    rm -f gradle.properties && \
    ./gradlew build && \
    mv /RahanSazehApiServer/build/libs/RahanSazeh-ApiServer-0.1.0.jar /RahanSazeh-ApiServer.jar && \
    cd / && \
    rm -rf /RahanSazehApiServer && \
    rm -rf /root/.gradle && \
    mkdir -p /RahanSazehApiServer && \
    mv /RahanSazeh-ApiServer.jar /RahanSazehApiServer/ && \
    chmod -R 755 /RahanSazehApiServer && \
    useradd -M -s /bin/false arcana

# heroku does not support EXPOSE
# EXPOSE 3000

WORKDIR /RahanSazehApiServer

# Run the image as a non-root user
USER arcana

CMD java -jar /RahanSazehApiServer/RahanSazeh-ApiServer.jar

```

The created docker image using above method is lightweight, around 20MBs which I have used to deploy my app to heroku cloud. 

1. I have deleted gradle cache to reduce image size. After building is done, we no longer need the gradle cache anymore! we can delete all ~300MB of downloaded packages and gradle blobs to reduce our final image size! this is done via `rm -rf /root/.gradle`
2. I have deleted java source files for the project after I've done building it! This is also crucial to keep our image size to the bare minimum level!


Like everything else, there are pros and cons to everything! The fact that we build our docker image straight from our git repository has pros and cons too! As this system enables us to safely build our spring (or java) application from our git repository, and can be integrated into our CI/CD or build systems, unlike NodeJs or Python or even Ruby requires lots of downloads for each build, and quite a time to download them all nasty Maven packages! But then again, If you are using a build server for this purpose (like mine) there is nothing to worry about! In fact, this is exactly what I needed!
