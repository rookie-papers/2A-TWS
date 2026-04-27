# SwiftAuth: Accountable Anonymous Authentication Mechanism Supporting Multi-Source Credential Aggregation and Non-Interactive Authorization

This repository provides the reference implementation of **2A-TWS**, a privacy-preserving and highly efficient service authorization system designed for UAV swarms.

For comprehensive performance evaluation, this repository also includes the implementations of three existing cryptographic schemes — **IhMA**, **DTACB**, and **NTAT** — which are used as comparative baselines in our experiments.

---

## ⚙️ Build Instructions

### 0️⃣ Install Required OS Dependencies

While our CMake configuration automatically fetches and builds core cryptographic libraries, the following system-level build tools and network dependencies must be installed first:

> `m4`: Strictly required for building the GMP library from source.
>
> `libboost-all-dev`: Required by `websocketpp` for network communications.
>
> Standard build tools: `git`, `cmake`, `python3`, `build-essential`.

```bash
sudo apt update
sudo apt install -y git cmake python3 build-essential m4 libboost-all-dev
```

### 1️⃣ Clone repository with submodules and build with CMake:

Using `-DCMAKE_BUILD_TYPE=Release` `-DBENCHMARK_ENABLE_WERROR=OFF` to avoid debug-mode overhead and prevent benchmark warnings from being treated as errors during compilation.
```bash
git clone --recurse-submodules https://github.com/rookie-papers/2A-TWS.git
cd 2A-TWS
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release -DBENCHMARK_ENABLE_WERROR=OFF ..
make -j
```

### 2️⃣ Running Benchmarks
Each scheme builds into an individual benchmark executable.

```bash
./DAT-TWS_exec
./DTACB_exec
./IhMA_exec
./NTAT_exec
```

## 📦 Dependencies

All dependencies are included as Git submodules:

| Library          | Source URL                                                                       |
|------------------|----------------------------------------------------------------------------------|
| GMP              | [github.com/rookie-papers/GMP](https://github.com/rookie-papers/GMP) (via fork with CMake support) |
| MIRACL Core      | [github.com/miracl/core](https://github.com/miracl/core)                         |
| Google Benchmark | [github.com/google/benchmark](https://github.com/google/benchmark)               |
| Websocketpp      | [github.com/zaphoyd/websocketpp](https://github.com/zaphoyd/websocketpp)               |

You don't need to install them manually — they are automatically configured and built with CMake.

---

## 🌐 Network Simulation Experiment

n addition to pure cryptographic benchmarking, this repository contains a high-fidelity network simulation environment located in the network_simulation directory. This environment emulates a distributed service authorization architecture using Linux network namespaces (netns) and traffic control (tc) to validate the protocol under realistic WAN conditions.

### 0️⃣ File Structure

The network_simulation directory is structured as follows:

```
network_simulation/
├── CMakeLists.txt          # CMake build configuration
├── scripts/                # Network automation and control scripts
│   ├── build_net.sh        # Builds virtual network topology (Namespaces, Bridges, Veth)
│   ├── config.env          # Config: issuer count, latency, bandwidth, loss, etc.
│   ├── run_issuers.sh      # Batch script to launch multiple Issuer processes
│   ├── tc_bandwidth.sh     # Applies bandwidth limit rules (Traffic Control)
│   ├── tc_latency.sh       # Applies network latency rules (Traffic Control)
│   ├── tc_loss.sh          # Applies packet loss simulation (Traffic Control)
│   └── teardown.sh         # Cleans up namespaces and restores network settings
└── src/                    # C++ source code implementation
    ├── regulator.cpp
    ├── issuer.cpp
    ├── user.cpp
    └── verifier.cpp
```

### 1️⃣ System Architecture and Network Topology

This experiment simulates the four key entities involved in the dynamic authorization of UAV swarms in our paper. Their functions are as follows:

> **Regulator:** The Trust Authority acting as the root of trust for system initialization, parameter generation.
>
> **Issuer:** Distributed cloud servers that process authorization requests, fetch user parameters from the Regulator, and issue anonymous credentials (Tags and Signatures) to the User.
>
> **User:** The UAV swarm requesting certificates concurrently from multiple Issuers and generating aggregated zero-knowledge proofs.
>
> **Verifier:** The ground station or service provider that verifies the aggregated proof before granting access.

To simulate the network environment and physical isolation described in the paper, each entity runs in its own independent Network Namespace (netns). This ensures that every node possesses a dedicated protocol stack, routing table, and virtual network interface.

The virtual network architecture is presented graphically below:

Here is a plain text version of the network topology diagram

```
Host (Linux Kernel)
                       +-----------------------+
                       |  br-dattws (10.0.0.1) |  <-- Central Virtual Bridge
                       +-----------+-----------+
                                   |
     +-----------------+-----------+-----------+-----------------+
     |                 |           |           |                 |
veth-reg-ns       veth-vf-ns  veth-usr-ns veth-iss1-ns      veth-issn-ns
     |                 |           |           |                 |
+----+----+       +----+----+ +----+----+ +----+-----+       +----+-----+
|Regulator|       |Verifier | |  User   | |Issuer 1  |  ...  |Issuer n  |
|10.0.0.10|       |10.0.0.20| |10.0.0.30| |10.0.0.101|       |10.0.0.10n|
+---------+       +---------+ +---------+ +----------+       +----------+                          |
```



Communication Workflow Summary:


> **Setup:** The Regulator generates public parameters. Issuers and the User fetch these parameters. 
> 
> **Issuance:** The User spawns multiple threads to request certificates from multiple Issuers concurrently. Each Issuer independently contacts the Regulator to fetch $H_u$, computes the partial credential, and returns it to the User.
> 
> **Verification:** The User aggregates the obtained credentials into a single zero-knowledge proof and submits it to the Verifier for authentication.


### 2️⃣ How to Run the Experiment

You can run the entire experiment sequence using the following commands. Ensure you have compiled the project first.

```bash
# Enter the simulation directory
cd build/network_simulation

# 1. Build the virtual network topology (Namespaces, Bridges, Veth pairs)
sudo ./scripts/build_net.sh

# 2. (Optional) Configure network conditions
# Specific parameters (e.g., NET_DELAY=50ms, NET_BANDWIDTH=128kbit) are defined in 'config.env'.
sudo ./scripts/tc_latency.sh     # Apply latency settings from config.env

# 3. Start the infrastructure entities in the background
sudo ip netns exec Regulator ./regulator &
sudo ip netns exec Verifier ./verifier &

# 4. Start all Issuer nodes in batch
sudo ./scripts/run_issuers.sh

# 5. Trigger concurrent issuance requests from the User
# Usage: ./user <num_issuers> <base_port> 
# "num_issuers" indicates the aggregation of several certificates for display.
sudo ip netns exec User ./user 5

# 6. Cleanup: Remove namespaces and restore network settings
sudo ./scripts/teardown.sh
```


## 🚀 Real Network Deployment

While the network_simulation directory focuses on single-machine virtualization, we also provide physical deployment scripts. This version allows entities to run on different physical devices (or Virtual Machines) and communicate via actual TCP/IP interfaces (e.g., Wi-Fi, Ethernet) over a real WAN.


### 0️⃣ File Structure

The `RTS-websocket` directory is structured as follows:

```text
protocol_simulation/
├── CMakeLists.txt          # CMake build configuration
├── scripts/                # Network control scripts for physical interfaces
│   ├── clean_tc.sh         # Restores normal network conditions (removes TC rules)
│   ├── config.env          # Global Config: issuer count, bandwidth, latency, etc.
│   ├── run_issuers.sh      # Batch script to launch multiple Issuer processes on ports
│   ├── stop_issuers.sh     # Utility script to safely terminate all Issuers
│   ├── tc_bandwidth.sh     # Applies bandwidth limits to physical NICs
│   ├── tc_latency.sh       # Applies network latency to physical NICs
│   └── tc_loss.sh          # Applies packet loss simulation to physical NICs
└── src/                    # C++ source code for network entities
    ├── regulator.cpp
    ├── issuer.cpp
    ├── user.cpp
    └── verifier.cpp
```

### 1️⃣ Deployment Architecture

This setup utilizes actual physical or virtual machines to test the protocol in a real distributed environment. A typical experimental setup (as used in our paper) includes three separate Virtual Machines:

> **Machine A** (192.168.206.128): Acts as the Cloud Center. Runs the Regulator and multiple Issuer instances (listening on different ports).
> 
> **Machine B** (192.168.206.129): Acts as the Ground Station. Runs the Verifier.
>
> **Machine C** (192.168.206.130): Simulates the UAV. Runs the User executable, which spawns concurrent threads to contact Machine A.


### 2️⃣ How to Run

Since this deployment involves multiple independent nodes, you must configure, build, and execute the programs on their respective hosts.
**1. Configuration**

You must manually update the IP addresses in the src/*.cpp files to match your actual local network environment before compiling. Ensure REGULATOR_IP, VERIFIER_IP, and DEFAULT_ISSUER_IP align with the physical machines hosting them.

**2. Compile on each machine**

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j
cd protocol_simulation
```

**3. Execute the corresponding commands on the respective devices.**

```bash
# ---------------------------------------------------------
# [Optional] Apply TC rules on Machine C (UAV) or Machine A
# Note: Ensure the interface name (e.g., ens33) is correct!
# ---------------------------------------------------------
sudo ./scripts/tc_latency.sh ens33

# ---------------------------------------------------------
# Step 1: Start Cloud Services on Machine A (192.168.206.128)
# ---------------------------------------------------------
./regulator &
./scripts/run_issuers.sh

# ---------------------------------------------------------
# Step 2: Start Verifier on Machine B (192.168.206.129)
# ---------------------------------------------------------
./verifier

# ---------------------------------------------------------
# Step 3: Trigger Authorization on Machine C (192.168.206.130)
# ---------------------------------------------------------
./user 5 8001

# ---------------------------------------------------------
# Cleanup: Remove TC rules and stop background services
# ---------------------------------------------------------
sudo ./scripts/clean_tc.sh  # Run on any machine with TC applied
./scripts/stop_issuers.sh   # Run on Machine A
```