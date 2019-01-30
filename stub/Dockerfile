FROM pactfoundation/pact-stub-server:latest
COPY pacts/pact.json  /app/pacts/
ENTRYPOINT ["/app/pact`-stub-server", "-d=/app/pacts", "-p=8080"]
EXPOSE 8080