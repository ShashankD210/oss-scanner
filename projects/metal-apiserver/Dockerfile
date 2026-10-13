FROM golang:1.27-trixie

COPY . /src
WORKDIR /src

RUN make server
RUN make test GO_TEST="-run TestTokenStoreTest"

# New reproducers and patches must also build with the network disabled.
ENV GOPROXY=off \
    GOSUMDB=off