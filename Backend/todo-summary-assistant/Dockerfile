# Stage 1: Build JAR with Maven
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Minimal runtime image (Non-root)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Run as non-root user
RUN adduser -D -u 1000 appuser
USER appuser

COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
