# Development

## Required Platforms

Before starting development, ensure all required platforms are installed. The table below lists each tool, its purpose, and where to download it.

| Platform                       | Purpose | Download                                                                                                                                                           |
|--------------------------------|---|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **IntelliJ IDEA**              | Primary IDE for API development (Java/Micronaut) | [jetbrains.com/idea](https://www.jetbrains.com/idea/download/) — download the Community (free) or Ultimate edition                                                 |
| **Java 17 (Eclipse Corretto)** | JDK required to build and run the API | [java corretto 17](https://docs.aws.amazon.com/corretto/latest/corretto-17-ug/downloads-list.html) — select your OS, choose JDK 17, download and run the installer |
| **Arduino IDE**                | IDE for writing and uploading PLC (Arduino) firmware | [arduino.cc/en/software](https://www.arduino.cc/en/software) — download the latest version for your OS                                                             |
| **MQTTX**                      | MQTT desktop client for publishing/subscribing to local broker topics during development | [mqttx.app](https://mqttx.app/) — download the desktop app for your OS                                                                                             |
| **Postman**                    | HTTP client for manually testing and exploring API endpoints locally | [postman.com/downloads](https://www.postman.com/downloads/) — download and install the desktop app                                                                 |
| **DBeaver**                    | GUI database client for inspecting and querying the local MySQL database | [dbeaver.io/download](https://dbeaver.io/download/) — download the Community edition for your OS                                                                   |
| **Docker Desktop**             | Runs the local infrastructure (MySQL, Mosquitto MQTT broker) via Docker Compose | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) — download and install for your OS                                           |
| **Node.js**                    | JavaScript runtime required to build and run the web frontend | [nodejs.org](https://nodejs.org/) — download the LTS version for your OS                                                                                           |
| **Git**                        | Version control — required to clone the repo and work with branches | [git-scm.com/downloads](https://git-scm.com/downloads) — download for your OS; macOS users can also run `xcode-select --install`                                   |
| **VS Code**                    | Lightweight editor, useful for web and PLC development as an alternative to IntelliJ | [code.visualstudio.com](https://code.visualstudio.com/) — download and run the installer                                                                           |

## Getting the Project

**Clone for the first time:**

```bash
git clone https://github.com/your-org/mkk.git
cd mkk
```

> Replace `https://github.com/your-org/mkk.git` with the actual repository URL found on the GitHub repository page (green **Code** button → copy HTTPS URL).

**Pull latest changes (already cloned):**

```bash
git pull --rebase
```

Run this from inside the project directory to fetch and merge the latest changes from the remote.

## IntelliJ IDEA Setup

Once you have the project cloned, open it in IntelliJ IDEA:

1. Open IntelliJ IDEA → **File → Open** → select the root project folder → click **OK**
2. Wait for the IDE to index the project and download Gradle dependencies (progress shown in the bottom status bar)
3. Set up project modules — watch the short guide: `/docs/general/modulesSetUp`
4. Set up run configurations — watch the short guide: `/docs/general/runSetUp`

## Running Locally

Start the local infrastructure (MySQL and Mosquitto MQTT broker) and application stack:

```bash
docker compose -f docker/docker-compose-infra.yaml up -d
```

The database and mqtt broker services run with the `infra` profile. No AWS credentials required.

## Build and Run

On how to build and run project locally (testing environment) you can watch this videos: `/docs/general/pirmadalis`, `/docs/general/antradalis`, `/docs/general/treciadalis`

Build and run the API:

```bash
cd api
./gradlew build
./gradlew run
```

Build and run the web frontend:

```bash
cd web
npm install
npm run build   # production build
npm run dev     # dev server with hot reload
```

## Tool for jwt token

JSON web token decoder: https://www.jwt.io/

## UUID generator for test device ID

UUID generator: https://www.uuidgenerator.net/version4

## Running Against AWS

For investigative tasks, you can run locally against AWS infrastructure using the tunnel and proxy config. See the [Tunnel](#tunnel) and [Proxy](#proxy) sections above.

Access is limited — no writes or deployments should be performed against the AWS environment this way.

# Deployment

Deployments are done exclusively via the **Build and Deploy** GitHub pipeline. There is no supported path for manual deployment by non-admin users.

## Device Setup

Devices are registered via GitHub actions.

After registration device id and certificate must be uploaded into flash memory.

Install tools for upload:
```bash
pip install esp-idf-nvs-partition-gen
pip install esptool
```

Run script:
```bash
./plc/tools/flash-device.sh <path> <usb>
```

# AWS

## Infra Setup

Infrastructure setup is an admin-only operation. Re-running against an existing environment may break infrastructure and incur unexpected costs.

Build the aws-cli image and run the interactive setup script:

```bash
docker build -f docker/Dockerfile.aws -t aws-cli .

docker run --rm -it --entrypoint bash \
    -v ~/.aws:/root/.aws \
    -v "$PWD:/mkk" \
    -w /mkk \
    -e AWS_PROFILE=mkk \
    -e AWS_PAGER="" \
    aws-cli \
    ./aws/scripts/setup-infra.sh eu-central-1
```

The script deploys all CloudFormation stacks interactively, one step at a time.

## Access

Contact the admin to receive an IAM user and access key. Add the credentials to `~/.aws/credentials` under the `mkk` profile:

```ini
[mkk]
aws_access_key_id = ...
aws_secret_access_key = ...
region = eu-central-1
```

Access is read-only and scoped to investigative tasks only. Direct deployment and infrastructure changes must go through GitHub pipelines.

## Tunnel

The tunnel provides SSH access to the AWS database for investigation purposes only.

Generate the tunnel compose file (fetches temporary SSH credentials from Lightsail, expires in ~1 hour):

```bash
docker run --rm -it --entrypoint bash \
    -v ~/.aws:/root/.aws \
    -v "$PWD:/mkk" \
    -w /mkk \
    -e AWS_PROFILE=mkk \
    aws-cli \
    ./aws/scripts/generate-tunnel.sh eu-central-1
```

Start the tunnel:

```bash
docker compose -f out/docker-compose-tunnel.yaml up -d
```

Re-run `generate-tunnel.sh` to refresh expired credentials.

## Proxy

The proxy config allows running the API locally against AWS infrastructure (database and MQTT). For investigation purposes only.

Generate `out/application-proxy.yaml` and IoT certificates in `out/iot/`:

```bash
docker run --rm -it --entrypoint bash \
    -v ~/.aws:/root/.aws \
    -v "$PWD:/mkk" \
    -w /mkk \
    -e AWS_PROFILE=mkk \
    -e AWS_PAGER="" \
    aws-cli \
    ./aws/scripts/generate-proxy.sh eu-central-1
```

Start the tunnel first (see above), then run the API with the proxy profile:

```bash
MICRONAUT_CONFIG_FILES=out/application-proxy.yaml MICRONAUT_ENVIRONMENTS=proxy ./gradlew run
```

The `out/` directory contains plaintext secrets — it is gitignored and must not be committed.

---

# GitHub

Pipelines are the only sanctioned path for deployment and administrative tasks. All workflows are manually triggered (`workflow_dispatch`).

| Pipeline | Purpose |
|---|---|
| **Build and Deploy** | Builds the API and web frontend, then deploys to Lightsail |
| **Setup IoT Device** | Provisions IoT certificates for a new device under a given tenant |
| **Setup Cognito User** | Creates a Cognito user with specified email, password, and group membership |
| **Start Lightsail Resources** | Starts the Lightsail container service and database |
| **Stop Lightsail Resources** | Stops the Lightsail container service and database |
| **Test AWS Connection** | Verifies GitHub OIDC authentication against AWS |

---
