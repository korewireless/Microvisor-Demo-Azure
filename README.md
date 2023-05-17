# Microvisor Azure MQTT Demo 1.0.0

This repo provides a basic demonstration of a user application capable of working with Microvisor’s MQTT communications system calls. It has no hardware dependencies beyond the Twilio Microvisor Nucleo Development Board.

It is based on the [FreeRTOS](https://freertos.org/) real-time operating system and which will run on the “non-secure” side of Microvisor. FreeRTOS is included as a submodule.

The [ARM CMSIS-RTOS API](https://github.com/ARM-software/CMSIS_5) is used as an intermediary between the application and FreeRTOS to make it easier to swap out the RTOS layer for another.

The application code files can be found in the [app_src/](app_src/) directory. The [ST_Code/](ST_Code/) directory contains required components that are not part of Twilio Microvisor STM32U5 HAL, which this sample accesses as a submodule. The `FreeRTOSConfig.h` and `stm32u5xx_hal_conf.h` configuration files are located in the [config/](config/) directory.

This demo can be used to work with Azure IoT.

We will:

- Create a device in an Azure IoT Hub with a Symmetric key
- Store the Symmetric in the secure configuration storage area on Microvisor cloud

This demo does not currently show the following flows.  They should be fully supported via Azure IoT's MQTT support.

- DPS provisioning of the device into a specific IoT Hub instance
- X509 self-signed or CA signed certificate authentication (see also the Amazon AWS demonstration for certificate authentication examples)

## Release Notes

Version 1.0.0 is the initial Azure demo.

## Actions

The code creates and runs four threads:

- A thread periodically toggles GPIO A5, which is the user LED on the [Microvisor Nucleo Development Board](https://www.twilio.com/docs/iot/microvisor/microvisor-nucleo-development-board).  This acts as a heartbeat to let you know the demo is working.
- A thread manages the network state of your application, requesting control of the network from Microvisor.
- A work thread which consumes events and dispatches them in support of the configuration loading and managed MQTT broker operations.
- An application thread which consumes data from an attached sensor (or demo source) and sends it to the work thread for publishing.

# Azure IoT Hub configuration

- Log into your Azure account at https://portal.azure.com/ , creating one if needed
- Create or select an IoT Hub instance within Azure.  This can be a Free Tier IoT Hub.

## Device configuration

- Within the IoT Hub, select Device Management -> Devices
- Select 'Add Device'
- For Device ID, enter exactly your Microvisor device identifier SID (starts with UV...)
- Select Authentication Type 'Symmetric Key', leave Auto-generate keys checked
- Click 'Save'

## Obtaining the Azure IoT Hub connection string

- You may need to click 'Refresh' for the device to show up in the device list after creating it
- Click on the device you created
- Locate 'Primary connection string' on the page and click the Show/Hide Field Contents icon to view it or the Copy to Clipboard icon to copy it
- Your connection string should look similar to: 'HostName=myhub.azure-devices.net;DeviceId=UV00000000000000000000000000000000;SharedAccessKey=QWhveSwgd29ybGQhCg=='

## Storing the connection string in Microvisor cloud

To facilitate your Microvisor device connecting, you wil need to provide the connection string to the device.  We recommend provisioning this as a device-scoped secret within Microvisor cloud so it is securely available to your device.

You'll need your Microvisor device SID, Account SID, and account Auth Token - all of which are available at https://console.twilio.com/

- Next, we will add configuration and secret items to Microvisor for this device

        # First, we'll set an environment variable with the connection string and other required info to make the curl a bit cleaner in the next step:

        export CONNECTION_STRING="HostName=myhub.azure-devices.net;DeviceId=UV00000000000000000000000000000000;SharedAccessKey=QWhveSwgd29ybGQhCg=="
        export TWILIO_ACCOUNT_SID=AC00000000000000000000000000000000
        export TWILIO_AUTH_TOKEN=.......
        export MV_DEVICE_SID=UV00000000000000000000000000000000

        curl --fail -X POST \
                --data-urlencode "Key=azure-connection-string" \
                --data-urlencode "Value=${CONNECTION_STRING}" \
                --silent https://microvisor.twilio.com/v1/Devices/${MV_DEVICE_SID}/Secrets \
                -u ${TWILIO_ACCOUNT_SID}:${TWILIO_AUTH_TOKEN}

# The demo

We use the connection string populated into the secrets store to generate shared access signature (SAS) tokens with an expiry.  We will be disconnected at the end of this expiration period and we'll generate a new SAS token and reconnect when this happens.

Azure IoT Hub has specific requirements for MQTT access, the demo in this repository uses the following topics:

Subscribes to: `devices/<device sid>/messages/devicebound/`

Publishes to: `devices/<device sid>/messages/events/`

Azure IoT MQTT requires MQTT 3.1.1, so be sure to configure the Microvisor managed MQTT client with protocol_version MV_MQTTPROTOCOLVERSION_V3_1_1 not MV_MQTTPROTOCOLVERSION_V5.

More details about Azure IoT's MQTT implementation can be found [in the Azure IoT documentation](https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-mqtt-support).

## Cloning the Repo

This repo makes uses of git submodules, some of which are nested within other submodules. To clone the repo, run:

```bash
git clone https://github.com/twilio/twilio-microvisor-azure-demo.git
```

and then:

```bash
cd twilio-microvisor-azure-demo
git submodule update --init --recursive
```

## Repo Updates

When the repo is updated, and you pull the changes, you should also always update dependency submodules. To do so, run:

```bash
git submodule update --remote --recursive
```

We recommend following this by deleting your `build` directory.

## Requirements

You will need a Twilio account. [Sign up now if you don’t have one](https://www.twilio.com/try-twilio).

You will also need a Twilio Microvisor [Nucleo Development Board](https://www.twilio.com/docs/iot/microvisor/microvisor-nucleo-development-board). These are currently only available to Beta Program participants: [Join the Beta](https://interactive.twilio.com/iot-microvisor-private-beta-sign-up?utm_source=github&utm_medium=github&utm_campaign=IOT&utm_content=MQTT_GitHub_Demo).

## Software Setup

This project is written in C. At this time, we only support Ubuntu 20.0.4. Users of other operating systems should build the code under a virtual machine running Ubuntu, or with Docker.

**Note** Users of unsupported platforms may attempt to install the Microvisor toolchain using [this guidance](https://www.twilio.com/docs/iot/microvisor/install-microvisor-app-development-tools-on-unsupported-platforms).

### With Docker

Build the image:

```shell
docker build --build-arg UID=$(id -u) --build-arg GID=$(id -g) -t mv-azure-demo-image .
```

Run the build:

```
docker run -it --rm -v $(pwd)/:/home/mvisor/project/ \
  --env-file env.list \
  --name mv-azure-demo mv-azure-demo-image
```

**Note** You will need to have exported certain environment variables, as [detailed below](#environment-variables).

Under Docker, the demo is compiled, uploaded and deployed to your development board. It also initiates logging — hit <b>ctrl</b>-<b>c</b> to break out to the command prompt.

Diagnosing crashes:

```
docker run -it --rm -v $(pwd)/:/home/mvisor/project/ \
  --env-file env.list \
  --name mv-azure-demo --entrypoint /bin/bash mv-azure-demo-image
```

To inspect useful info, to start with PC and LR:
```
gdb-multiarch project/build/app/mv-azure-demo.elf
info symbol <...>
```

### Without Docker

#### Libraries and Tools

Under Ubuntu, run the following:

```bash
sudo apt install gcc-arm-none-eabi binutils-arm-none-eabi \
  git curl build-essential cmake libsecret-1-dev jq openssl
```

#### Twilio CLI

Install the Twilio CLI. This is required to view streamed logs and for remote debugging. You need version 4.0.1 or above.

**Note** If you have already installed the Twilio CLI using *npm*, we recommend removing it and then reinstalling as outlined below. Remove the old version with `npm remove -g twilio-cli`.

```bash
wget -qO- https://twilio-cli-prod.s3.amazonaws.com/twilio_pub.asc | sudo apt-key add -
sudo touch /etc/apt/sources.list.d/twilio.list
echo 'deb https://twilio-cli-prod.s3.amazonaws.com/apt/ /' | sudo tee /etc/apt/sources.list.d/twilio.list
sudo apt update
sudo apt install -y twilio
```

Close your terminal window or tab, and open a new one. Now run:

```bash
twilio plugins:install @twilio/plugin-microvisor
```

### Environment Variables

Running the Twilio CLI and the project's [deploy script](./deploy.sh) — for uploading the built code to the Twilio cloud and subsequent deployment to your Microvisor Nucleo Board — uses the following Twilio credentials stored as environment variables. They should be added to your shell profile:

```bash
export TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
export TWILIO_AUTH_TOKEN=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
export MV_DEVICE_SID=UVxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

You can get the first two from your Twilio Console [account dashboard](https://console.twilio.com/).

Enter the following command to get your target device’s SID and, if set, its unqiue name:

```bash
twilio api:microvisor:v1:devices:list
```

## Build and Deploy the Application

Run:

```bash
./deploy.sh --log
```

This will compile, bundle and upload the code, and stage it for deployment to your device. If you encounter errors, please check your stored Twilio credentials.

The `--log` flag initiates log-streaming.

## Choosing which application to build

Multiple applications are supported in the demo. Right now the two applications implemented are

    - `dummy` - send dummy temperature data once a second
    - `temperature` - send real temperature read from TH02 sensor

You can choose what application to build by providing `--application` argument to `deploy.sh`

```bash
./deploy.sh --application temperature --log
```

By default `dummy` application is built.

## View Log Output

You can start log streaming separately — for example, in a second terminal window — with this command:

```bash
./deploy.sh --logonly
```

For more information, run

```bash
./deploy.sh --help
```

## Remote Debugging

This release supports remote debugging, and builds are enabled for remote debugging automatically. Change the value of the line

```
set(ENABLE_REMOTE_DEBUGGING 1)
```

in the root `CMakeLists.txt` file to `0` to disable this.

Enabling remote debugging in the build does not initiate a GDB session — you will have to do this manually. Follow the instructions in the [Microvisor documentation](https://www.twilio.com/docs/iot/microvisor/microvisor-remote-debugging) **Private Beta participants only**

This repo contains a `.gdbinit` file which sets the remote target to localhost on port 8001 to match the Twilio CLI Microvisor plugin remote debugging defaults.

### Remote Debugging Encryption

Remote debugging sessions are encrypted. To generate keys, add the `--gen-keys` switch to the deploy script call, or generate your own keys — see the documentation linked above for details.

Use the `--public-key-path` and `--private-key-path` options to either specify existing keys, or to specify where you would like script-generated keys to be stored. By default, keys will be stored in the `build` directory so they will not be inadvertently push to a public git repo:

```
./deploy.sh --private-key /path/to/private/key.pem --public-key /path/to/public/key.pem
```

You will need to pass the path to the private key to the Twilio CLI Microvisor plugin to decrypt debugging data. The deploy script will output this path for you.

## Copyright and Licensing

The sample code and Microvisor SDK is © 2022, Twilio, Inc. It is licensed under the terms of the [Apache 2.0 License](./LICENSE).

The SDK makes used of code © 2021, STMicroelectronics and affiliates. This code is licensed under terms described in [this file](https://github.com/twilio/twilio-microvisor-hal-stm32u5/blob/main/LICENSE-STM32CubeU5.md).

The SDK makes use [ARM CMSIS](https://github.com/ARM-software/CMSIS_5) © 2004, ARM. It is licensed under the terms of the [Apache 2.0 License](./LICENSE).

[FreeRTOS](https://freertos.org/) is © 2021, Amazon Web Services, Inc. It is licensed under the terms of the [Apache 2.0 License](./LICENSE).

[cifra](https://github.com/ctz/cifra), created by Joseph Birr-Pixton <jpixton@gmail.com> is licensed under [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
