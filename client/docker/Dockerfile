FROM docker.localregistry.com/library/java-base
# copy arthas
COPY --from=hengyunabc/arthas:latest /opt/arthas /opt/arthas
ARG JAR_FILE
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app.jar"]