FROM huydng/github-service:latest

RUN mkdir -p "config"
COPY config /home/jboss/config

ENTRYPOINT ["java", "-Djava.util.logging.manager=org.jboss.logmanager.LogManager", "-jar", "quarkus-run.jar"]