FROM openjdk:8-jdk-alpine
EXPOSE 8080
ARG JAR_FILE=./target/*.jar
ADD ./target/*.jar app.jar
ENTRYPOINT ["java","-jar","/app.jar"]