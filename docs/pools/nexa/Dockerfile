FROM ubuntu:latest as builder

ARG NEXA_SHORT_VERSION=1.4.0.1
ARG NEXA_VERSION=1.4.0.1


ENV DEBIAN_FRONTEND=noninteractive
RUN apt update && \
    apt install -y curl && \
    apt clean && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /build

RUN curl -LO https://bitcoinunlimited.info/nexa/${NEXA_SHORT_VERSION}/nexa-${NEXA_VERSION}-linux64.tar.gz && \
    tar xavf nexa-${NEXA_VERSION}-linux64.tar.gz && \
    rm nexa-${NEXA_VERSION}-linux64.tar.gz && \
    mv nexa-${NEXA_VERSION} nexa

FROM ubuntu:latest

COPY --from=builder /build/nexa /nexa

ENV HOME=/nexa
ENV PATH="${PATH}:/nexa/bin"

EXPOSE 7228 7227

WORKDIR /nexa

ARG GROUP_ID=1001
ARG USER_ID=1001

RUN groupadd -g ${GROUP_ID} nexa && \ 
    useradd -u ${USER_ID} -g nexa -d /nexa nexa

RUN chown -R nexa:nexa /nexa

ENV DEBIAN_FRONTEND=noninteractive
RUN apt update && \
    apt install -y --no-install-recommends libatomic1 jq && \
    apt clean && \
    rm -rf /var/lib/apt/lists/*

USER nexa

ENTRYPOINT ["nexad"]
