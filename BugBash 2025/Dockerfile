# Build container
FROM alpine:3.21 AS build

# Setup environment
RUN mkdir -p /data
WORKDIR /data

# Install native dependencies
RUN apk add zig git

# Get sources
RUN git clone https://github.com/tigerbeetle/tigerbeetle.git
WORKDIR /data/tigerbeetle

# Build the release
COPY . .
RUN zig build fuzz:build

# Extract the release
RUN mkdir -p /out
RUN cp ./zig-out/bin/fuzz /out/fuzz

# Release container
FROM alpine:3.21

# Add the release
COPY --from=build /out/fuzz /usr/local/bin/fuzz

CMD ["sh", "-c", "(while true; do /usr/local/bin/fuzz lsm_scan; done) & (while true; do FUZZ_PID=$(pgrep -f '/usr/local/bin/fuzz lsm_scan' | tr '\\n' ','); echo \"Fuzzing@$(date '+%Y-%m-%d %H:%M:%S') FUZZER_PID=${FUZZ_PID%,} ...\"; sleep 1; done)"]