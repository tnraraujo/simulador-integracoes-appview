FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY build/libs/pega-bpm-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8082
ENTRYPOINT ["java", "-jar", "app.jar"]
