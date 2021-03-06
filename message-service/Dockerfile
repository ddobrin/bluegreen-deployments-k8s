# Multistage docker build that splits the spring boot app into three layers
# Layer with all the libraries which are likely to be the same between
# mutliple builds of the container which means better layer caching
# Layer with the app META-INF
# layer with the actualy app code

# in the first stage we extract the executable boot jar into /app
# so that the following stage can copy it
FROM adoptopenjdk:11-jdk-hotspot AS exploded
COPY target/message-service-1.0.0.jar /app.jar
RUN mkdir /app \
  && cd /app \
  && jar xf /app.jar

# in this stage we produce the three layer boot image
FROM adoptopenjdk:11-jre-hotspot
COPY --from=exploded /app/BOOT-INF/lib /app/lib
COPY --from=exploded /app/META-INF /app/META-INF
COPY --from=exploded /app/BOOT-INF/classes /app
VOLUME /tmp
WORKDIR /app
EXPOSE 8080
USER nobody:nobody

# use shell from of entry point and expect k8s manifest to set the JAVA_OPTS env var
ENTRYPOINT java $JAVA_OPTS -cp /app:/app/lib/* com.example.messageservice.MessageServiceApplication

# exec from of entry point below can not do variable subsittion
# ENTRYPOINT ["java","-XshowSettings:vm","-XX:MaxRAMPercentage=75.0","-cp","/app:/app/lib/*","com.example.messageservice.MessageServiceApplication"]
