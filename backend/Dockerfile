# Build stage
FROM eclipse-temurin:21-jdk AS build

WORKDIR /app

COPY . .

RUN chmod +x mvnw
RUN ./mvnw clean package -DskipTests


# Runtime stage
FROM eclipse-temurin:21-jre

WORKDIR /app

# Create dedicated non-root user
RUN groupadd --gid 10001 appgroup \
    && useradd --uid 10001 --gid 10001 \
       --create-home \
       --shell /usr/sbin/nologin \
       appuser

COPY --from=build /app/target/*.jar backend.jar

RUN mkdir -p /app/logs \
    && chown -R 10001:10001 /app

USER 10001:10001

EXPOSE 8088

ENTRYPOINT ["java", "-jar", "backend.jar"]


# # Build stage
# FROM eclipse-temurin:21-jdk AS build

# WORKDIR /app

# COPY . .

# RUN chmod +x mvnw
# RUN ./mvnw clean package -DskipTests


# # Runtime stage
# FROM eclipse-temurin:21-jre

# WORKDIR /app

# # Create non-root user
# RUN groupadd appgroup \
#     && useradd \
#        --create-home \
#        --gid appgroup \
#        --shell /usr/sbin/nologin \
#        appuser

# COPY --from=build /app/target/*.jar backend.jar

# RUN mkdir -p /app/logs \
#     && chown -R appuser:appgroup /app

# USER appuser

# EXPOSE 8088

# ENTRYPOINT ["java", "-jar", "backend.jar"]

# # Build stage
# FROM eclipse-temurin:21-jdk AS build

# WORKDIR /app
# COPY . .
# RUN chmod +x mvnw
# RUN ./mvnw clean package -DskipTests


# # Runtime stage
# FROM eclipse-temurin:21-jre

# WORKDIR /app

# RUN groupadd --gid 1000 appuser \
#     && useradd --uid 1000 --gid 1000 \
#        --create-home \
#        --shell /usr/sbin/nologin appuser

# COPY --from=build /app/target/*.jar backend.jar

# RUN mkdir -p /app/logs \
#     && chown -R 1000:1000 /app

# USER 1000:1000

# EXPOSE 8088

# ENTRYPOINT ["java", "-jar", "backend.jar"]
