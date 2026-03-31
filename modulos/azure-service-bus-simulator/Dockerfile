FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY build/libs/azure-service-bus-simulator-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8088
ENTRYPOINT ["java", "-jar", "app.jar"]
