# Raw MQTT to Azure IoT Hub (in Python) without device SDK
A simulated device that implements cloud-to-device and device-to-cloud messaging as well as direct methods over MQTT without using the Azure IoT Device SDK.

This project is using Paho to talk MQTT to Azure IoT Hub.

# WARNING
EVERYTHING HERE IS HIGHLY EXPERIMENTAL.

## Generate the SAS token with azure-cli

First, install the Azure IoT extension: https://github.com/azure/azure-iot-cli-extension#installation

Then generate the SAS key:
```
az iot hub generate-sas-token \
    --hub-name YOUR_IOT_HUB_NAME \
    --device-id YOUR_DEVICE_ID \
    --key-type primary \
    --duration 3600
```

## Send cloud to device message
```
az iot device c2d-message send \
    --hub-name poorlyfundedskynet \
    --device-id yolo --data \
    '{"hey device": "this is a cloud to device message"}'
```

## Invoke direct method
```
az iot hub invoke-device-method \
    --hub-name poorlyfundedskynet
    --device-id yolo \
    --method-name hey \
    --method-payload '{"hey device": "this is a direct method invocation"}'
```