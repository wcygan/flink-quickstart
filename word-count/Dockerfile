# ===== Build Stage =====
FROM maven:3.9.3 AS builder

# Set the working directory inside the container
WORKDIR /usr/src/app

# Copy the pom.xml file first to leverage Docker's cache
COPY pom.xml .

# Download Maven dependencies without building the project
RUN mvn dependency:go-offline

# Copy the source code after dependencies to ensure that changes in source code
# don't invalidate the cached dependencies layer
COPY src ./src

# Build the project
RUN mvn clean package

# ===== Final Stage =====
FROM flink:1.16.2-scala_2.12-java11

# Create a directory for the target if it doesn't exist
RUN mkdir -p /code/target

# Copy the built artifacts from the builder stage
COPY --from=builder /usr/src/app/target/ /code/target

# Create a symbolic link for Flink web uploads
RUN ln -s /code/target $FLINK_HOME/flink-web-upload

# Set the default command
CMD ["help"]
