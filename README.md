# MemStream

MemStream is a lightweight C++ data-pipeline engine, version **2.0.0**, designed to construct and execute distributed processing pipelines using independent processing nodes. The nodes communicate through **ZeroMQ sockets** and can be composed into a **Directed Acyclic Graph (DAG)** for controlled data flow and processing.

### Core Components

- **RandomGeneratorNode** – Generates and publishes pseudo-random data samples on demand.
- **TcpSocketNode** – Receives and republishes pipeline data through a TCP socket. The default TCP port is **9001**.
- **UdpSocketNode** – Receives and republishes pipeline data through a UDP socket. The default UDP port is **9002**.
- **controller.py** – Provides runtime control over the `RandomGeneratorNode`, allowing the output route to be dynamically selected between TCP and UDP.

# Build & Run

### 1. Build the Docker Image

Build the versioned Docker image using:

```bash
docker build -t cpp-pipeline:2.0.0 .
```

### 2. Extract the Debian Package from the Docker Image

Create a temporary container from the image, copy the generated Debian package to the host, and remove the temporary container:

```bash
CID=$(docker create cpp-pipeline:2.0.0)
docker cp "$CID":/workspace/build/cpp-pipeline-2.0.0-Linux.deb .
docker rm "$CID"
```

### 3. Install the Package on the Host

The package can be installed on an Ubuntu/Debian-based host using:

```bash
sudo apt install ./cpp-pipeline-2.0.0-Linux.deb
```

Alternatively, install it using `dpkg` and resolve any missing dependencies with:

```bash
sudo dpkg -i cpp-pipeline-2.0.0-Linux.deb && sudo apt -f install
```

### 4. Start the Pipeline Engine

The pipeline engine is started by providing the path to the JSON configuration file:

```bash
cp_engine /etc/cpp-pipeline/config.json
```

# Engine

The **Engine** is responsible for initializing and orchestrating the configured pipeline.

At startup, the engine:

- Loads the pipeline definition from the specified JSON configuration file.
- Validates the configured **Directed Acyclic Graph (DAG)**.
- Resolves the relationships between the configured nodes.
- Initializes the individual pipeline nodes.
- Passes the `--input` and `--output` parameters to the corresponding nodes.
- Manages the overall execution of the configured data pipeline.

The `--input` and `--output` parameters define the data interfaces through which individual nodes receive and publish pipeline data.

# Nodes

Each pipeline node follows a **producer/consumer-style concurrency architecture**.

The node separates socket I/O from processing logic using two execution contexts:

- The **main thread** is responsible for pulling data from and pushing data to the associated socket interfaces.
- The **worker thread** performs the node-specific data processing.
- The worker thread communicates processing results or status notifications back to the main thread.

This architecture allows socket communication and data processing to operate independently while maintaining controlled synchronization between the two execution contexts.

# Helper Scripts

## 1. controller.py

`controller.py` provides runtime control over the `RandomGeneratorNode` output routing.

It instructs the `RandomGeneratorNode` to route future generated samples to the **TCP outlet**, **UDP outlet**, or **both outlets**.

### Usage

```bash
python3 controller.py TCP|UDP|BOTH
```

Supported modes:

- `TCP` – Route generated samples to the TCP outlet.
- `UDP` – Route generated samples to the UDP outlet.
- `BOTH` – Route generated samples to both TCP and UDP outlets.

## 2. udp_client.py

`udp_client.py` acts as a UDP client for the `UdpSocketNode`.

It connects to the UDP endpoint exposed by `UdpSocketNode` and receives the data published on the port configured in `config.json`.

Ensure that the port configured in `udp_client.py` matches the UDP port specified in `config.json`.

### Usage

```bash
python3 udp_client.py
```

## 3. ws_client.py

`ws_client.py` acts as a client for the TCP endpoint exposed by `TcpSocketNode`.

It connects to the TCP endpoint and receives data published by the `TcpSocketNode` using the port configured in `config.json`.

Ensure that the port configured in `ws_client.py` matches the TCP port specified in `config.json`.

### Usage

```bash
python3 ws_client.py
```