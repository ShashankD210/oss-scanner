FROM espressif/idf:v5.2

COPY . /src
WORKDIR /src

RUN . $IDF_PATH/export.sh && \
    idf.py set-target esp32 && \
    idf.py build

RUN . $IDF_PATH/export.sh && \
    python3 tools/test_build_health.py --target esp32

RUN python3 tools/test_shell_host_integration.py
