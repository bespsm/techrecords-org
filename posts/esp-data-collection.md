---
title: Architectural Evolution. Lessons from Merging AWS and ESP32 Development
taxonomy:
    category: Projects
    post_tag:
        - DEV
        - software
        - architecture
        - IT
        - ESP32
        - AWS
        - cloud
---

> This article reviews the architecture of cloud-based visualization of the data from ESP32 microcontroller. My experience may help other developers/architects avoid the problems I encountered during the development process. It contains the links to source code, HOWTOs locally and in AWS and lessons learnt.


in Summer 2024 I finished one after another trainings, namely [*IoT Application Development with ESP32*](https://www.udemy.com/course/iot-application-development-with-the-esp32-using-the-esp-idf/) from Udemy and *"Deploying Serverless Application on AWS with Terraform"* provided by my former employer.
Have you had a feeling after passing a training *"hm, that was good but I want to practice it!"*. At least I had one. So I decided make a project where I can leverage the new knowledge. I set two simple goals:
* visualize the data from ESP32 as time series, namely temperature, humidity, RSSI level.
* add it to EPS32 embedded web-server and make it as WEB service the cloud.

![Wiring diagram of ESP32 with DHT22 sensor](/_images/esp-dc-circuit.png "Wiring diagram of ESP32 with DHT22 sensor") {.wp-post-image}

![Assembled circuit](/_images/esp-dc-assembly-look.jpg "Assembled circuit") {.wp-post-image}

## Visualization in ESP32 embedded web-server

No issues were there. I extended original [IoT course's repository](https://github.com/kevinudemy/udemy_esp32) by adding **chart.js** library and sensor chart to embedded ESP32 web-server. My fork with extensions and detailed description is located [here](https://github.com/bespsm/esp-data-collection-soc).

![Visualization in ESP32 embedded web-server](/_images/esp-dc-local-visual.png "Visualization in ESP32 embedded web-server") {.wp-post-image}

## WEB service visualization on AWS. Initial Software Architecture

The idea was to deploy [Grafana](https://grafana.com/) as UI on EC2, add AWS IoT Thing as MQTT Broker, use DynamoDB as data storage, set up API Gateway for accessing data over HTTP and add few lambdas to glue things together.

![Architecture of AWS-based Visualization. Version 1.0](/_images/esp-dc-design-v1.0.png "Architecture of AWS-based Visualization. Version 1.0") {.wp-post-image}

At that moment I didn't really study Grafana and its data sources and naively thinking that it supports data fetching from REST API or DynamoDB out of box.

## Revised Software Architecture: Attempt Two

During the development I decided to simplify design by moving from lambdas and instead use a [python script](https://github.com/bespsm/esp-data-collection-srv/blob/main/script) for forwarding MQTT messages to DynamoDB and use DynamoDB data source in Grafana for fetching time series data. A part from that I replaced AWS IoT MQTT Broker by Eclipse Mosquitto MQTT Broker which I installed on the same EC2 instance where Grafana was. Route53 is used to have a static hostname for ESP32.

![Visualization on AWS. Version two](/_images/esp-dc-design-v2.0.png "Architecture of AWS-based Visualization. Version 2.0") {.wp-post-image}

At the stage when I tried to connect Grafana and DynamoDB I realized that the current architecture would require additional expenses. I avoided it and come up with the final architecture design (down below).

Anyways I decided to upload this implementation partially finished (with no Grafana connection). Even more, there are 2 configurations of MQTT communications on the server side, namely using user/pass authentication and using communication over TLS. __Pay attention: SoC flashware is configured for TLS communication only.__

## Final Software Architecture

Only at that point I started to study Grafana and its data sources. It seemed that Prometheus together with Prometheus Pushagateway was most common choice for collecting time series data over REST.

![Visualization on AWS. Version three](/_images/esp-dc-design-v3.0.png "Architecture of AWS-based Visualization. Version 3.0") {.wp-post-image}

![Grafana Dashboard look](/_images/esp-dc-grafana-dash.png "Grafana Dashboard") {.wp-post-image}

Here are the links to the final architecture: [Terraform config](https://github.com/bespsm/esp-data-collection-tf/tree/main/prometheus-grafana), [SoC flashware](https://github.com/bespsm/esp-data-collection-SoC) and [Grafana config](https://github.com/bespsm/esp-data-collection-srv/tree/main/grafana_cfg) (is deployed by Terraform). *The final solution is not focused on security aspects. This one is left on the used user.*

## Lessons Learnt

After going over 3 architecture iterations, I summarize the outcome in 2 points:
* The simpler the architecture, the better. I went down from 7 elements to 5 at the final stage.
* Get to know all the components and the interfaces between them in your architecture. I would safe a lot of time avoiding useless implementation if I'd study Grafana beforehand.

## How to reproduce

__Flash ESP32__
* assemble ESP32 with a DHT22 sensor, according to wiring diagram
* clone [the course's repo with my extenstions](https://github.com/bespsm/esp-data-collection-soc)
* read the READE.md and adapt the code to your needs
* build and flash to your hardware

__AWS deployment of server side__
* clone [Terraform configs repo](https://github.com/bespsm/esp-data-collection-tf)
* read the READE.md
* adapt [techrecords_grafana.tfvars](https://github.com/bespsm/esp-data-collection-tf/blob/main/prometheus-grafana/techrecords_grafana.tfvars) to your needs
```
cd esp-data-collection-tf/prometheus-grafana/
terraform apply  -var-file=techrecords_grafana.tfvars
```
Grafana should be accessible over the port 3000 and EC2 IP (or your subdomain name from "techrecords_grafana.tfvars")

__Local deployment of server side__ (tested on Ubuntu 22.04)
* invoke following commands:
```
git clone https://github.com/bespsm/esp-data-collection-srv.git
cd esp-data-collection-srv
docker compose up -d
```
* go to [grafana local page](http://localhost:3000)



## Useful Links

1. [IoT Application Development with ESP32](https://www.udemy.com/course/iot-application-development-with-the-esp32-using-the-esp-idf/)
2. [How to Set Up a Mosquitto MQTT Broker Securely](https://medium.com/gravio-edge-iot-platform/how-to-set-up-a-mosquitto-mqtt-broker-securely-using-client-certificates-82b2aaaef9c8)
