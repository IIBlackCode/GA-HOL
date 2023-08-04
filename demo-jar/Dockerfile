FROM gradle:jdk17 as build
COPY --chown=gradle:gradle . /home/gradle/src
WORKDIR /home/gradle/src
RUN gradle build --no-daemon

FROM azul/zulu-openjdk-alpine:17-jre-latest
EXPOSE 1337
RUN mkdir /app
COPY --from=build /home/gradle/src/build/libs/*-SNAPSHOT.jar /app/app.jar
CMD ["java", "-jar", "/app/app.jar"]