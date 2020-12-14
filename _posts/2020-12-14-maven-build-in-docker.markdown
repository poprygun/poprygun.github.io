---
layout: post
title:  "In case you dont have Maven installed on your development workstation..."
date:   2020-12-14 12:54:49 -0500
categories: maven docker
---
Handy tips to build java source when Maven is not installed.  You can bring your own image with pre-configured Maven.

```dockerfile
FROM adoptopenjdk/openjdk11:alpine-slim
RUN apk add --no-cache curl tar bash
ARG MAVEN_VERSION=3.6.3
ARG USER_HOME_DIR="/root"
RUN mkdir -p /usr/share/maven && \
curl -fsSL http://apache.osuosl.org/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz | tar -xzC /usr/share/maven --strip-components=1 && \
ln -s /usr/share/maven/bin/mvn /usr/bin/mvn
ENV MAVEN_HOME /usr/share/maven
ENV MAVEN_CONFIG "$USER_HOME_DIR/.m2"
ENV MAVEN_OPTS="-XX:+TieredCompilation -XX:TieredStopAtLevel=1"
ENTRYPOINT ["/usr/bin/mvn"]
RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app
COPY pom.xml /usr/src/app
RUN mvn -T 1C install -D spring-boot.repackage.skip=true && rm -rf target
COPY src /usr/src/app/src
```

Spring Boot Maven Plugin needs `main` class to package sources into Spring Boot application.  At this point, we are only interested in pulling all dependencies into image, not creating the app, therefore `spring-boot.repackage.skip=true`

Create the image

```bash
docker build -t ashumilov/mvn-builder:latest .
```

To package the application in offline mode

```bash
docker run -it --rm -v $(pwd)/target:/usr/src/app/target ashumilov/mvn-builder package -T 1C -o -Dmaven.test.skip=true
```
