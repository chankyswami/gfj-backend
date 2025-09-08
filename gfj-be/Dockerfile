# Use an official JDK 21 image as a parent image
FROM docker.io/eclipse-temurin:21-jdk

# Set the working directory in the container
WORKDIR /opt

# Copy the JAR file into /opt
COPY target/*.jar app.jar

# Expose at 8081
EXPOSE 8081

# Start application
CMD ["java", "-jar", "app.jar"]
