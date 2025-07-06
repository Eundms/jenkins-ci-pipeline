FROM openjdk:17
ARG IDLE_PROFILE
ENV ENV_IDLE_PROFILE=$IDLE_PROFILE
COPY app.jar /app.jar
ENTRYPOINT ["java", "-Dspring.profiles.active=${ENV_IDLE_PROFILE}", "-jar", "/app.jar"]
