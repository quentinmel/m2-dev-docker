FROM alpine:3.22

RUN --mount=type=secret,id=token \
    cat /run/secrets/token > /tmp/token-used

CMD ["sh"]