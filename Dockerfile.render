FROM maven:3-openjdk-11 AS builder

# Install required dependencies
RUN apt-get update && apt-get install -y \
    wget \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY . .

# Create necessary directories
RUN mkdir -p temp sql target

# Download required files
RUN wget https://github.com/RESOStandards/web-api-commander/releases/download/current-version/web-api-commander.jar -O temp/web-api-commander.jar || \
    curl -L https://github.com/RESOStandards/web-api-commander/releases/download/current-version/web-api-commander.jar --output temp/web-api-commander.jar

RUN wget https://raw.githubusercontent.com/RESOStandards/web-api-commander/main/src/main/resources/RESODataDictionary-1.7.metadata-report.json -O RESODataDictionary-1.7.metadata-report.json || \
    curl -L https://raw.githubusercontent.com/RESOStandards/web-api-commander/main/src/main/resources/RESODataDictionary-1.7.metadata-report.json --output RESODataDictionary-1.7.metadata-report.json

# Download PostgreSQL JDBC driver
RUN wget https://jdbc.postgresql.org/download/postgresql-42.6.0.jar -O temp/postgresql-42.6.0.jar || \
    curl -L https://jdbc.postgresql.org/download/postgresql-42.6.0.jar --output temp/postgresql-42.6.0.jar

# Generate SQL scripts
RUN java -jar temp/web-api-commander.jar --generateReferenceDDL --useKeyNumeric > sql/reso-reference-ddl-dd-1.7.numeric-keys.sql

# Build the project
RUN if command -v gradle &> /dev/null; then \
        gradle build && \
        cp build/libs/RESOservice-1.0.war ./target/core.war; \
    else \
        mvn package && \
        mv ./target/RESOservice-1.0.war ./target/core.war; \
    fi

# Copy the metadata report to the target directory
RUN cp RESODataDictionary-1.7.metadata-report.json ./target/

FROM tomcat:9

# Copy the built artifacts from the builder stage
COPY --from=builder /app/target/core.war /usr/local/tomcat/webapps/
COPY --from=builder /app/target/RESODataDictionary-1.7.metadata-report.json /usr/local/tomcat/webapps/
COPY --from=builder /app/temp/postgresql-42.6.0.jar /usr/local/tomcat/lib/

# Set environment variables for remote debugging if needed
ENV JPDA_ADDRESS="*:8000"
ENV JPDA_TRANSPORT="dt_socket"

# Set environment variables for database connection
ENV SQL_HOST=${SQL_HOST:-localhost}
ENV SQL_USER=${SQL_USER:-postgres}
ENV SQL_PASSWORD=${SQL_PASSWORD:-postgres}
ENV SQL_DB_DRIVER=${SQL_DB_DRIVER:-org.postgresql.Driver}
ENV SQL_CONNECTION_STR=${SQL_CONNECTION_STR:-jdbc:postgresql://${SQL_HOST}/postgres}
ENV CERT_REPORT_FILENAME=${CERT_REPORT_FILENAME:-RESODataDictionary-1.7.metadata-report.json}

# Expose ports
EXPOSE 8080
EXPOSE 8000

# Start Tomcat with JPDA enabled
CMD ["catalina.sh", "jpda", "run"] 