FROM maven:3.6.3-jdk-8-slim AS build
WORKDIR /app

COPY pom.xml .
COPY src ./src

RUN mvn clean package -DskipTests