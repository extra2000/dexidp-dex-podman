Production Deployment
=====================

Example how to deploy Dex for single-instance production environment.

Clone repositories
------------------

.. code-block:: bash

    mkdir ~/extra2000
    cd ~/extra2000
    git clone https://github.com/extra2000/dexidp-dex-podman.git
    git clone https://github.com/extra2000/dexidp-dex.git dexidp-dex-podman/src/dexidp-dex

Then, ``cd`` into project root directory:

.. code-block:: bash

    cd dexidp-dex-podman

Build Dex Image
---------------

From the project root directory, ``cd`` into ``src/dexidp-dex/`` and then build:

.. code-block:: bash

    cd src/dexidp-dex/
    podman build -t extra2000/dexidp/dex .

Preparing MySQL Database
------------------------

Create table and user:

.. code-block:: mysql

    CREATE DATABASE dexdb;
    CREATE USER 'dex' IDENTIFIED BY 'abcde12345';
    GRANT ALL ON `dexdb`.* TO 'dex'@'%';

Deploy Dex
----------

From the project root directory, ``cd`` into ``deployment/production/dex``:

.. code-block:: bash

    cd deployment/production/dex

Create config files and fix permissions:

.. code-block:: bash

    cp -v configmaps/dex.yaml{.example,}
    cp -v configs/config.yaml{.example,}
    chmod og+r ./configs/config.yaml

Create pod file:

.. code-block:: bash

    cp -v dexidp-dex-pod.yaml{.example,}

For SELinux platform, label the following files to allow to be mounted into container:

.. code-block:: bash

    chcon -R -v -t container_file_t ./configs

Create SELinux security policy:

.. code-block:: bash

    cp -v selinux/dexidp_dex_podman.cil{.example,}

Load SELinux security policy:

.. code-block:: bash

    sudo semodule -i selinux/dexidp_dex_podman.cil /usr/share/udica/templates/base_container.cil

Verify that the SELinux module exists:

.. code-block:: bash

    sudo semodule --list | grep -e "dexidp_dex_podman"

Deploy Dex:

.. code-block:: bash

    podman play kube --configmap configmaps/dex.yaml --seccomp-profile-root ./seccomp dexidp-dex-pod.yaml

Create systemd files to run at startup:

.. code-block:: bash

    mkdir -pv ~/.config/systemd/user
    cd ~/.config/systemd/user
    podman generate systemd --files --name dexidp-dex-pod
    systemctl --user enable pod-dexidp-dex-pod.service container-dexidp-dex-pod-srv01.service

Testing
-------

Set the following configurations in ``configs/config.yaml``:

.. code-block:: yaml

    issuer: http://127.0.0.1:5556/dex

    storage:
    type: mysql
    config:
        host: 127.0.0.1
        port: 3306
        database: dexdb
        user: dex
        password: abcde12345
        ssl:
        mode: "false"

    logger:
    level: "debug"
    format: "text" # can also be "json"

    web:
    http: 127.0.0.1:5556

    telemetry:
    http: 127.0.0.1:5558

    grpc:
    addr: 127.0.0.1:5557

    staticClients:
    - id: example-app
        redirectURIs:
        - 'http://127.0.0.1:5555/callback'
        name: 'Example App'
        secret: ZXhhbXBsZS1hcHAtc2VjcmV0

    connectors:
    - type: mockCallback
        id: mock
        name: Example

    enablePasswordDB: true

    staticPasswords:
    - email: "admin@example.com"
        hash: "$2a$10$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
        username: "admin"
        userID: "08a8684b-db88-4b73-90a9-3cd1661f5466"

Build example app and then run. From project's root directory, execute the following command:

.. code-block:: bash

    cd src/
    chcon -R -v -t container_file_t dexidp-dex
    podman run --network host -it --rm -v ./dexidp-dex:/opt/dex:rw --workdir /opt/dex docker.io/golang:1.17.7 bash
    make examples
    /opt/dex/bin/example-app --listen http://127.0.0.1:5555 --issuer http://127.0.0.1:5556/dex
