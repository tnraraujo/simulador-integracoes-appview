FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY build/libs/postgresql-simulator-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8087
ENTRYPOINT ["java", "-jar", "app.jar"]
