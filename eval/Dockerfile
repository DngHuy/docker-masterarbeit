ARG evaluationDate

FROM huydng/eval-service:latest

RUN mkdir -p "projects"
COPY projects /home/jboss/projects

ENTRYPOINT [ "/opt/jboss/container/java/run/run-java.sh" ]