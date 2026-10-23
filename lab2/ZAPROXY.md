# OWASP ZAP with Podman and Webswing

This guide runs the ZAP desktop interface in a browser using Webswing.

## Contents

- [Run ZAP with Linux Package](#run-zap-with-linux-package)
- [Run ZAP with Podman](#run-zap-with-podman)
  - [Open the ZAP interface](#open-the-zap-interface)
  - [Stop ZAP](#stop-zap)
- [Configure Firefox](#configure-firefox)
- [References](#references)


## Run ZAP with Linux Package

1. Download the Linux Package from the official repository <https://www.zaproxy.org/download/>

2. Extract the package
    ```bash
    tar -zxvf <package name>.tar.gz
    ```

3. Run the binary:

    ```bash
    cd <package name>
    ./zap.sh
    ``` 

## Run ZAP with Podman

1. Start the official stable ZAP image with Webswing. The Webswing interface is
   available on port `3000` on your computer, and the ZAP proxy listens on port
   `8008`. Both ports are bound to localhost so they are not exposed to other
   computers on the network.

   ```bash
   podman run -it --rm --name zaproxy \
     -p 127.0.0.1:3000:8080 \
     -p 127.0.0.1:8008:8090 \
     ghcr.io/zaproxy/zaproxy:stable zap-webswing.sh
   ```

### Open the ZAP interface

Open <http://localhost:3000/zap/> in your browser. Webswing displays the ZAP
desktop interface there.

For the WebGoat exercise, start WebGoat as described in the lab's
[README](README.md). When entering its address in ZAP or Firefox, use
`http://host.containers.internal:8080/webgoat/attack`. From inside the
container, `host.containers.internal` refers to the host computer where
WebGoat's port `8080` is published.

### Stop ZAP

Close the Webswing browser tab, then press `Ctrl+C` in the terminal running
Podman.


## Configure Firefox
Follow the same steps defined in the [README](README.md)



## References

- [Zap Downloads](https://www.zaproxy.org/download/)

- [ZAP Docker images](https://www.zaproxy.org/docs/docker/about/)
- [ZAP Configuring proxies](https://www.zaproxy.org/docs/desktop/start/proxies/)
- [Podman installation](https://podman.io/docs/installation)