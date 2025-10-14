# Part 1 — TeamServer & Architecture

This is **Part 1** of a short series on **C2TeamServer** — a modular Command-and-Control framework I maintain at `https://github.com/maxDcb/C2TeamServer`.
In this post I describe the **TeamServer** core: how it is built, how it communicates with clients, how listeners and modules are organised, and which configuration knobs control its behaviour. This is the architectural foundation; later posts will dive into listeners, the GUI, and implants.

---

## 🧭 High-Level Goals

**TeamServer** was built with two core principles in mind:

* **🧩 Modularity** — The architecture allows you to add new **listeners** (responsible for handling communication channels with implants) and **modules** (which define executable tasks) with minimal or no modifications to the core logic.
* **🚀 Extensibility** — Capabilities can be expanded dynamically through **loadable modules** (`.so`), enabling new functionality to be integrated without recompiling or restarting the server.

---

## 🛠️ uild & dependency management — Conan + CMake

The project is composed of several submodules that provide reusable code used across the repository (for example by the beacon):

* **[C2Core](https://github.com/maxDcb/C2Core)** — one of the most important components; it contains the implementation of modules, beacons and listeners.
* **[libDns](https://github.com/maxDcb/libDnsCommunication)** — DNS communication support.
* **[libSocketHandler](https://github.com/maxDcb/libSocketHandler)** — TCP communication helpers.
* **[libSocks5](https://github.com/maxDcb/libSocks5)** — SOCKS5 implementation.

Third-party utilities used by the project:

* **[cpp-base64](https://github.com/ReneNyffenegger/cpp-base64)** — base64 encoding/decoding.
* **[Donut](https://github.com/TheWover/donut)** — a pivotal project that inspired parts of this work; used for `assemblyExec` functionality and others.
* **[nlohmann/json](https://github.com/nlohmann/json)** — the JSON library used by the project (not included as a submodule).
                                                                             
To keep builds "friendly" (it's C++...) I use **Conan** for dependency management and **CMake** for the build.

Example `conanfile.txt` (excerpt):

```ini
[requires]
grpc/1.72.0
protobuf/5.27.0
spdlog/1.15.3
cpp-httplib/0.20.1
openssl/3.5.1

[layout]
cmake_layout

[generators]
CMakeDeps
```

### Key libraries

**gRPC**
gRPC is the RPC framework that provides the TeamServer’s communication backbone with the client. It handles remote procedure calls via protobufs. Using gRPC gives automatic client/server codegen for multiple languages, simplifies streaming, and enforces a clear protobuf-defined contract between server and GUI. Note: it’s not used for the implant transport to keep the implant dependency minimal and lightweight.

**Protobuf**
Protocol Buffers is the serialization format and schema language used to define every message exchanged over gRPC.

**OpenSSL**
OpenSSL supplies the TLS/SSL stack used to secure gRPC connections and any other TLS endpoints (HTTPS listeners).

Typical local workflow:

```bash
# Install dependencies and generate cmake toolchain
mkdir build
cd build
cmake .. -DCMAKE_PROJECT_TOP_LEVEL_INCLUDES=./conan_provider.cmake
make -j4
```

### Why Conan?

Conan lets me **pin exact versions** of dependencies like gRPC, OpenSSL, Abseil, and Protobuf, ensuring **consistent and reproducible builds** across all environments.

With a proper profile and binary cache (ConanCenter, Artifactory, or local), it **avoids long rebuilds** by reusing precompiled artifacts, cutting build times from minutes to seconds and making CI/CD pipelines more reliable.

---

## ⚙️ Configuration

TeamServer is configured with a JSON file (`TeamServerConfig.json`) that contains settings for logging, directory layout, network and connection configuration, TLS credentials, and listener definitions. The file defines where artifacts live, how the server advertises itself (domain / IP / interface), gRPC server parameters, and the TLS files used for secure communications. Listener-specific options are stored under their respective keys (e.g., `ListenerHttpConfig`) so the server can start and manage endpoints dynamically.

```json
{
  "LogLevel": "info",

  "TeamServerModulesDirectoryPath": "../TeamServerModules/",
  "LinuxModulesDirectoryPath": "../LinuxModules/",
  "WindowsModulesDirectoryPath": "../WindowsModules/",
  "LinuxBeaconsDirectoryPath": "../LinuxBeacons/",
  "WindowsBeaconsDirectoryPath": "../WindowsBeacons/",
  "ToolsDirectoryPath": "../Tools/",
  "ScriptsDirectoryPath": "../Scripts/",

  "DomainName": "superdomain.io",
  "ExposedIp": "",
  "IpInterface": "eth0",

  "ServerGRPCAdd": "0.0.0.0",
  "ServerGRPCPort": "50051",
  "ServCrtFile": "server.crt",
  "ServKeyFile": "server.key",
  "RootCA": "rootCA.crt",

  "ListenerHttpConfig": {}
}
```

### Field-by-field explanation

* **`LogLevel`**
  Controls verbosity of server logs (`trace`, `debug`, `info`, `warning`, `error`). Use `info` for normal operation and `debug` during troubleshooting.

* **Directory layout**

  * **`TeamServerModulesDirectoryPath`** — modules compiled and used by the TeamServer (shared libs, server-side helpers).
  * **`LinuxModulesDirectoryPath` / `WindowsModulesDirectoryPath`** — platform-specific modules compiled for the beacon on Linux/Windows respectively. These are the binaries the server can serve to implants if needed.
  * **`LinuxBeaconsDirectoryPath` / `WindowsBeaconsDirectoryPath`** — prebuilt beacon binaries or templates for each platform (used to build droppers/implant artifacts).
  * **`ToolsDirectoryPath`** — common tools (examples: helper utilities).
  * **`ScriptsDirectoryPath`** — packaging/build scripts (examples: PowerShell or shell scripts used for dropper generation).

* **Exposed network / dropper generation**

  * **`DomainName`** — a public domain used for HTTP(S) listeners and for generating droppers that point to the server (e.g., `updates.example.com`).
  * **`ExposedIp`** — raw IP to advertise in droppers if you prefer IP over domain.
  * **`IpInterface`** — interface name used for testing or when you want the server to auto-detect the bind address from a specific NIC (useful in multi-NIC dev hosts).
    These fields are used by droppers and build scripts to embed the correct callback address.

* **gRPC & TLS**

  * **`ServerGRPCAdd` / `ServerGRPCPort`** — address and port on which the gRPC server listens (usually `0.0.0.0` to listen on all interfaces).
  * **`ServCrtFile` / `ServKeyFile`** — filesystem paths to the server TLS certificate and private key used by gRPC and HTTPS listeners.
  * **`RootCA`** — CA certificate used to validate client certificates for mTLS (or to generate bundles for clients). If you require mTLS, make sure this file contains the CA that signed client certs.

* **`ListenerHttpConfig` (and other listener sections)**
  Listener-specific configuration lives under separate objects so individual transports can define their own options (paths, domain fronting, TLS overrides, jitter/backoff settings, etc.).

---

## 📡 RPC backbone — gRPC with TLS

TeamServer exposes a **gRPC API** for clients. gRPC gives us:

* Strong typing (protobuf) → fewer parsing bugs.
* Built-in streaming for long-running sessions and large file transfers.
* Language-agnostic clients (the client could be C/C++, Go, Python, Nim...).

All gRPC endpoints are protected by **TLS**. Certificates are configured in the TeamServer config (see earlier).

All procedures and messages between client and server are defined here: `libs/libGrpcMessages/src/TeamServerApi.proto`

```proto
// Interface exported by the server.
service TeamServerApi 
{
  rpc GetListeners(Empty) returns (stream Listener) {}
  rpc AddListener(Listener) returns (Response) {}
  rpc StopListener(Listener) returns (Response) {}

  rpc GetSessions(Empty) returns (stream Session) {}
  rpc StopSession(Session) returns (Response) {}

  rpc GetHelp(Command) returns (CommandResponse) {}
  rpc SendCmdToSession(Command) returns (Response) {}
  rpc GetResponseFromSession(Session) returns (stream CommandResponse) {}

  rpc SendTermCmd(TermCommand) returns (TermCommand) {}
}
```

```proto
message Response 
{
  ...
}

// message related to all Listener procedures
message Listener 
{
  string listenerHash = 1;
  string type = 2;
  int32 port = 3;
  string ip = 4;
  string project = 6;
  string token = 7;
  string domain = 8;
  int32 numberOfSession = 5;
  string beaconHash = 9;
}

message Session 
{
  ...
}

message Command 
{
  ...
}

message CommandResponse 
{
  ...
}

message TermCommand 
{
  ...
}
```

### Common gRPC commands

* `GetListeners`:
  **Purpose:** List all configured/active listeners on the TeamServer.
  **When called:** GUI requests the current listener inventory to populate a UI or assert state.

* `AddListener`:
  **Purpose:** Create and start a new listener using the provided configuration (name, type, options).
  **When called:** When an operator wants to expose a new endpoint dynamically.

* `StopListener`:
  **Purpose:** Stop and remove a running listener.
  **When called:** When an operator wants to disable or reconfigure a listener.

* `GetSessions`:
  **Purpose:** List all active implant sessions currently registered on the TeamServer.
  **When called:** GUI requests the current session inventory to populate a UI or assert state.

* `StopSession`:
  **Purpose:** Terminate a specific implant session.
  **When called:** When an operator wants to close a session.

* `GetHelp`:
  **Purpose:** Retrieve help text or usage information for a specific command or module.
  **When called:** When the operator needs to display contextual command documentation.

* `SendCmdToSession`:
  **Purpose:** Send a task or command to a specific session for execution.
  **When called:** When an operator issues a task (e.g., run shell command, execute module).

* `GetResponseFromSession`:
  **Purpose:** Stream the responses, output, or progress from a session after commands are sent.
  **When called:** When the GUI wants to view real-time results.

* `SendTermCmd`:
  **Purpose:** Send a command that interacts with the TeamServer control plane (not a listener or session).
  **When called:** When an operator wants to manage the TeamServer (examples: `reloadModules`, `getBeaconBinary`, `putIntoUploadDir`, `addCredential`, `getCredential`, `socks`).

---

## 🛜 Listeners: pluggable transport channels

A **listener** is a named network endpoint that accepts implant connections.
Listeners are defined in the [C2Core](https://github.com/maxDcb/C2Core/tree/master/listener) submodule, as some of them are also used by implants to enable peer-to-peer communication between implants.

Each listener has:

* `type` — transport (`http`, `https`, `dns`, `tcp`, `smb`, `namedpipe`, …).
* `params` — IP, port, interface, domain, etc.
* `optional param` — transport-specific settings (TLS profile, domain fronting options, jitter settings, etc.).

The implementation uses standard C++ inheritance to abstract the communication layer from message and session handling:

```c++
class Listener
{
public:
    Listener(const std::string& param1, const std::string& param2, const std::string& type);
    ...
};

class ListenerHttp : public Listener
{
public:
    ListenerHttp(const std::string& ip, int localport, const nlohmann::json& config, bool isHttps=false);
    ...
};
```

Listeners are managed by the TeamServer using a list of running listeners: `m_listeners`. The `IsPrimary` parameter indicates whether the listener is hosted by the TeamServer or by a beacon.

```c++
else if (type == ListenerHttpType)
{
    json configHttp = m_config["ListenerHttpConfig"];
    
    int localPort = listenerToCreate->port();
    string localHost = listenerToCreate->ip();
    std::shared_ptr<ListenerHttp> listenerHttp = make_shared<ListenerHttp>(localHost, localPort, configHttp, false);
    int ret = listenerHttp->init();
    if (ret > 0)
    {
        listenerHttp->setIsPrimary();
        m_listeners.push_back(std::move(listenerHttp));

        m_logger->info("AddListener Http {0}:{1}", localHost, std::to_string(localPort));
    }
    else
    {
        m_logger->error("Error: AddListener Http {0}:{1}", localHost, std::to_string(localPort));
    }   
}
```

```log
[2401-06-06 02:48:20.594] [TeamServer] [info] AddListener Https 0.0.0.0:8443
[2401-06-06 02:48:20.595] [Listener_https_8443_OKAHxey7] [info] uriFileDownload /uri/used/to/deliver/files/
[2401-06-06 02:48:20.595] [Listener_https_8443_OKAHxey7] [info] downloadFolder ../www
[2401-06-06 02:48:20.595] [Listener_https_8443_OKAHxey7] [info] uri /Endpoint/to/communicate/with/httpListener/endpoint1.php
[2401-06-06 02:48:20.595] [Listener_https_8443_OKAHxey7] [info] uri /Endpoint/to/communicate/with/httpListener/endpoint2.php
[2401-06-06 02:48:20.595] [Listener_https_8443_OKAHxey7] [info] uri /OneMore/endpoint3.php
...
```

The same `m_listeners` list is used to access sessions, which represent implants connected to a listener. By enumerating sessions inside each listener, we can retrieve data sent by implants.

```c++
grpc::Status TeamServer::GetSessions(grpc::ServerContext* context, const teamserverapi::Empty* empty, grpc::ServerWriter<teamserverapi::Session>* writer)
{
    m_logger->trace("GetSessions");

    for (int i = 0; i < m_listeners.size(); i++)
    {
        m_logger->trace("Listener {0}", m_listeners[i]->getListenerHash());

        int nbSession = m_listeners[i]->getNumberOfSession();
        for (int kk = 0; kk < nbSession; kk++)
        {
            std::shared_ptr<Session> session = m_listeners[i]->getSessionPtr(kk);

            m_logger->trace("Session {0} From {1} {2}", session->getBeaconHash(), session->getListenerHash(), session->getLastProofOfLife());
            ...
        }
    }
}
```

```c++
grpc::Status TeamServer::SendCmdToSession(grpc::ServerContext* context, const teamserverapi::Command* command, teamserverapi::Response* response)
{
    ...
}

int TeamServer::handleCmdResponse()
{
    ...
}
```

Listeners are also responsible for adding a layer of encryption between the beacon and the listener. The key is embedded in the listener and the beacon and therefore not configurable at runtime; however, keys are XOR-encrypted at compile time to avoid appearing as plain strings in static analysis:

```c++
// XOR encrypted at compile time, so doesn't appear as a string literal
constexpr std::string_view _KeyTraficEncryption_ = "ComEncryptionKey";
constexpr std::string_view mainKeyConfig = ".CRT$XCL";

// compile time encryption
constexpr std::array<char, 29> _EncryptedKeyTraficEncryption_ = compileTimeXOR<29, 8>(_KeyTraficEncryption_, mainKeyConfig);

Listener::Listener(const std::string& param1, const std::string& param2, const std::string& type)
{   
    ...
    // decrypt key
    std::string keyDecrypted(std::begin(_EncryptedKeyTraficEncryption_), std::end(_EncryptedKeyTraficEncryption_));
    std::string key(mainKeyConfig);
    XOR(keyDecrypted, key);
    ...
}

bool Listener::handleMessages(const std::string& input, std::string& output)
{
    std::string data = base64_decode(input);
    XOR(data, m_key);
    ...
}
```

Finally, the communication between the **Beacon** and the **Listener** is twofold: there’s the **transport channel** (e.g., HTTP/HTTPS) and the **message layer**. In our case, we use a [JSON-based implementation](https://github.com/maxDcb/C2Core/tree/master/modules/ModuleCmd/C2Message.hpp) combined with a **bundling mechanism** to structure and transmit data efficiently.

```c++
// simple message -> a task for example
class C2Message
{
public:
    C2Message() { }
};

// multiple messages coming from one single beacon
class BundleC2Message
{
public:
    BundleC2Message() { }

private:
    std::vector<std::unique_ptr<C2Message>> m_c2Messages;
};

// multiple Bundles coming from multiple beacons possibly chained in a peer-to-peer setup
class MultiBundleC2Message
{
public:
    MultiBundleC2Message() { }

    void ParseFromArray(const char* data, int size)
    {
        std::string input(data, size);
        nlohmann::json my_json;
        ...
    }

    void SerializeToString(std::string* output)
    {
        nlohmann::json agregator;
        ...
        *output = agregator.dump();
    }

    BundleC2Message* add_bundlec2messages()
    {
        ...
    }

private:
    std::vector<std::unique_ptr<BundleC2Message>> m_bundleC2Messages;
};
```

**Conclusion:**
Listeners are modular in the sense that they decouple the communication channel from the messaging system. They are used both by the TeamServer and by the Beacon to achieve the same objective: distribute messages and handle sessions.

---

## 🧩 Modules — shared libraries (.so) for message generation & tasks

To keep TeamServer lean and extensible, functional logic is packaged as **shared libraries** (`.so`) that the TeamServer can load at runtime. Paired libraries (`.so` for Linux and `.dll` for Windows) are built for the Beacon side as well. The paired concept acknowledges that the functionality needed by the TeamServer and the implant differ: modules compiled for the TeamServer exclude exploit logic, while modules compiled for the Beacon exclude server-only helpers to reduce detection surface.

Benefits:

* Add / replace modules without rebuilding TeamServer or restarting the process.
* Allow implants to `FetchModule` on demand (reduces initial implant size and exposure).

### Module contract

Each module inherits from [ModuleCmd](https://github.com/maxDcb/C2Core/blob/master/modules/ModuleCmd/ModuleCmd.hpp) so the TeamServer and the Beacon both handle a list of `ModuleCmd` instances without needing to know implementation details:

```c++
class ModuleCmd
{
public:
    ModuleCmd(const std::string& name, unsigned long long hash=0);
    ~ModuleCmd();

    std::string getName();
    unsigned long long getHash();
    virtual std::string getInfo() = 0;
    virtual int init(std::vector<std::string>& splitedCmd, C2Message& c2Message) = 0;
    virtual int initConfig(const nlohmann::json &config) { return 0; };
    virtual int process(C2Message& c2Message, C2Message& c2RetMessage) = 0;
    virtual int followUp(const C2Message &c2RetMessage) { return 0; };
    virtual int errorCodeToMsg(const C2Message &c2RetMessage, std::string& errorMsg) { return 0; };
    virtual int recurringExec (C2Message& c2RetMessage) { return 0; };
    virtual int osCompatibility () { return OS_NONE; };

protected:

private:

};

class Cat : public ModuleCmd
{
public:
    Cat();
    ~Cat();

    std::string getInfo();

    int init(std::vector<std::string>& splitedCmd, C2Message& c2Message);
    int process(C2Message& c2Message, C2Message& c2RetMessage);
    int errorCodeToMsg(const C2Message &c2RetMessage, std::string& errorMsg);
    int osCompatibility()
    {
        return OS_LINUX | OS_WINDOWS;
    }

private:

};
```

At startup the TeamServer scans the configured `TeamServerModulesDirectoryPath` and registers available modules. Modules also include metadata (name, hash, usage information) so the server can enumerate and display them.

Example module load in C++:

```cpp
TeamServer::TeamServer(const nlohmann::json& config) 
{
    ...
    m_logger->info("TeamServer module directory path {0}", m_teamServerModulesDirectoryPath.c_str());
    try 
    {
        for (const auto& entry : fs::recursive_directory_iterator(m_teamServerModulesDirectoryPath)) 
        {
            if (fs::is_regular_file(entry.path()) && entry.path().extension() == ".so") 
            {
                m_logger->info("Trying to load {0}", entry.path().c_str());

                void *handle = dlopen(entry.path().c_str(), RTLD_LAZY);

                if (!handle) 
                {
                    m_logger->warn("Failed to load {0}", entry.path().c_str());
                    continue;
                }

                std::string funcName = ...

                m_logger->info("Looking for constructor function {0}", funcName);

                constructProc construct = (constructProc)dlsym(handle, funcName.c_str());
                if (construct == NULL) 
                {
                    ...
                }

                ModuleCmd* moduleCmd = construct();

                std::unique_ptr<ModuleCmd> moduleCmd_(moduleCmd);
                m_moduleCmd.push_back(std::move(moduleCmd_));
            }
        }
    }
    ...
}
```

```log                                                                                                     
└─$ ./TeamServer                                                                                                      
[2401-06-06 10:08:07.362] [TeamServer] [info] TeamServer logging initialized at debug level                        
[2401-06-06 10:08:07.364] [TeamServer] [debug] TeamServer logging initialized at debug level           
[2401-06-06 10:08:07.365] [TeamServer] [info] Authentication enabled for 2 user(s) using credentials file: auth_credentials.json                                                                                                            
[2401-06-06 10:08:07.366] [TeamServer] [debug] TeamServer module directory path ../TeamServerModules/                                                                                                                                       
[2401-06-06 10:08:07.367] [TeamServer] [debug] Trying to load ../TeamServerModules/libEvasion.so             
[2401-06-06 10:08:07.367] [TeamServer] [debug] Looking for construtor function EvasionConstructor
[2401-06-06 10:08:07.367] [TeamServer] [debug] Module libEvasion.so loaded
[2401-06-06 10:08:07.367] [TeamServer] [debug] Trying to load ../TeamServerModules/libScreenShot.so
[2401-06-06 10:08:07.368] [TeamServer] [debug] Looking for construtor function ScreenShotConstructor
[2401-06-06 10:08:07.368] [TeamServer] [debug] Module libScreenShot.so loaded 
...
[2401-06-06 10:08:07.404] [TeamServer] [debug] Module libChisel.so loaded                    
[2401-06-06 10:08:07.404] [TeamServer] [info] Loaded 38 TeamServer module(s) from ../TeamServerModules/
[2401-06-06 10:08:07.493] [TeamServer] [info] Team Server listening on 0.0.0.0:50051 
```

We can request reloading of modules if one is added or updated:

```c++
grpc::Status TeamServer::SendTermCmd(grpc::ServerContext* context, const teamserverapi::TermCommand* command, teamserverapi::TermCommand* response)
{
    ...
    else if (instruction == ReloadModulesInstruction)
    {
        m_logger->info("Reloading TeamServer modules from directory: {0}", m_teamServerModulesDirectoryPath.c_str());

        // Clear previously loaded modules
        m_moduleCmd.clear();
        ...
    }
}
```

Modules embed only the **strictly required components**, which helps **reduce exposure to EDR scanning** when used by the Beacon. This is achieved by **not compiling unnecessary parts** and by **minimizing the number of strings** in the final delivered artifact.

```c++
std::string Cat::getInfo()
{
    std::string info;
#ifdef BUILD_TEAMSERVER
    info += "Cat Module:\n";
    info += "Read and display the contents of a file from the victim machine.\n";
    ....
#endif
    return info;
}

int Cat::init(std::vector<std::string> &splitedCmd, C2Message &c2Message)
{
#if defined(BUILD_TEAMSERVER) || defined(BUILD_TESTS)
    if (splitedCmd.size() >= 2 )
    {
        ....
        c2Message.set_instruction(splitedCmd[0]);
        c2Message.set_inputfile(inputFile);
    }
    else
    {
        c2Message.set_returnvalue(getInfo());
        return -1;
    }
#endif
    return 0;
}

int Cat::errorCodeToMsg(const C2Message &c2RetMessage, std::string& errorMsg)
{
#ifdef BUILD_TEAMSERVER
    int errorCode = c2RetMessage.errorCode();
    if (errorCode > 0)
    {
        if (errorCode == ERROR_OPEN_FILE)
            errorMsg = "Failed: Couldn't open file";
    }
#endif
    return 0;
}
```

Beacon modules are sent to the target using the **`LoadModule`** command, and it’s the **responsibility of the Beacon** to manage a list of loaded modules on its side. Modules can also be used on the **server side** to construct messages destined for their **Beacon counterpart** and to trigger the corresponding module’s execution.

```c++
int TeamServer::prepMsg(const std::string& input, C2Message& c2Message, bool isWindows)
{
    ...
    for (auto it = m_moduleCmd.begin(); it != m_moduleCmd.end(); ++it)
    {
        if (toLower(instruction) == toLower((*it)->getName())) 
        {
            splitedCmd[0] = (*it)->getName();
            res = (*it)->init(splitedCmd, c2Message);
            isModuleFound = true;
        }
    }

    // send c2Message via the session -> listener chain.
    ...
}
```

**Conclusion**
Modules are **modular by design**, meaning they can be **hot-loaded** dynamically on both the **server side** and the **Beacon side** without requiring a full rebuild or redeployment.

This architectural choice provides several advantages:

* 🧩 **Flexibility:** New capabilities can be added or updated at runtime, allowing operators to adapt quickly during an engagement.
* 🕵️ **Stealth:** By loading only the required functionality when needed, the overall Beacon footprint remains minimal, reducing the risk of detection by EDR or antivirus solutions.
* ⚡ **Maintainability:** Each module is self-contained, making it easier to maintain, test, and update without impacting the rest of the framework.
* 🔐 **Security:** Isolating capabilities into modules limits exposure — both in terms of attack surface and forensic visibility.

This modular and hot-swappable approach is a **core design principle** of the C2 framework, ensuring that both the TeamServer and the Beacon remain **lightweight**, **adaptable**, and **operationally efficient**.

---

## 🔀 SOCKS5 & Proxy Support

In addition to classic listeners (HTTP, TCP, DNS, etc.), the TeamServer also supports a **SOCKS5 mode**, enabling operator traffic proxying and pivoting via connected implants. When a SOCKS5 is active, the TeamServer will accept incoming SOCKS5 client connections (e.g. from your attack workstation) and then forward the data through normal listeners to the implant living on the target network.

The flow is:

0. SOCKS5 server is started on the TeamServer, and is bind to an existing beacon.
1. An operator connects to the TeamServer’s SOCKS5, sending the SOCKS5 handshake and subsequent TCP relay requests.
2. The TeamServer resolves or accepts the target host/port request and wraps that into a `C2Message` message forwarded to the realted session.
3. The implant receives the request and opens a local socket to the destination, then relays data bi-directionally between that socket and the TeamServer via the same messaging channel.
4. Data is streamed transparently, so the operator sees target-side traffic as though they were directly connected.

This SOCKS5 functionality leverages the same listener/message infrastructure used for other tasks: request encryption, and multiplexing are reused. Because it's integrated at the listener/session layer, it inherits the same as other messages.

```c++
grpc::Status TeamServer::SendTermCmd(grpc::ServerContext* context, const teamserverapi::TermCommand* command,  teamserverapi::TermCommand* response)
{
    ...
    else if(instruction == SocksInstruction_)
	{
        ...
            else if(cmd == "bind")
			{
				if(!m_isSocksServerRunning)
				{
					...
				}
				if(m_isSocksServerBinded)
				{
					...
				}
				if(splitedCmd.size()==3)
				{
					std::string beaconHash = splitedCmd[2];
					for (int i = 0; i < m_listeners.size(); i++)
					{
						int nbSession = m_listeners[i]->getNumberOfSession();
						for(int kk=0; kk<nbSession; kk++)
						{
							std::shared_ptr<Session> session = m_listeners[i]->getSessionPtr(kk);
							std::string hash = session->getBeaconHash();
							if (hash.find(beaconHash) != std::string::npos && !session->isSessionKilled()) 
							{
                                // Set the socksListener to the Listener witht the connection to the beacon
								m_socksListener = m_listeners[i];

                                // Set the socksSession to the session related to the target beacon
								m_socksSession = m_listeners[i]->getSessionPtr(kk);

                                // Start the socks thread
								m_socksThread = std::make_unique<std::thread>(&TeamServer::socksThread, this);

                                ...
                            }
                        }
                    }
                }
            }
    }
}


void TeamServer::socksThread()
{
    ...
    while(m_isSocksServerBinded)
    {

		C2Message c2Message = m_socksListener->getSocksTaskResult(m_socksSession->getBeaconHash());
        
        ...

                else if(state == SocksState::RUN)
				{
					m_logger->trace("Socks5 run {}", id);

					dataIn="";
					if(c2Message.instruction() == Socks5Cmd && c2Message.cmd() == RunCmd && c2Message.pid() == id)
					{
						m_logger->debug("Socks5 {}: data received from beacon", id);

						dataIn=c2Message.data();
					
                        // Connected to the tool using the socks server, ex: proxychain rdesktop...
						int res = tunnel->process(dataIn, dataOut);

						m_logger->debug("Socks5 process, rec {}, dataIn {}, to send {}", res, dataIn.size(), dataOut.size());

                        ...
                        else
						{
							m_logger->debug("Socks5 send data to beacon");

							C2Message c2MessageToSend;
							...

							if(!c2MessageToSend.instruction().empty())
								m_socksListener->queueTask(m_socksSession->getBeaconHash(), c2MessageToSend);
						}
                        ...

}
```

SOCKS5 support relies on a **custom [implementation](https://github.com/maxDcb/libSocks5)** that was developed as an independent library.
This design allows the SOCKS5 component to be **reused outside of the C2 framework**, for example in standalone tooling, operator workstations, or other network pivoting setups.

The library handles the SOCKS5 protocol stack: including handshake, authentication, and data forwarding and is designed to integrate cleanly with the TeamServer’s session and listener model. By separating the SOCKS5 logic from the C2 core, the implementation remains **modular**, **testable**, and **easy to maintain**.


---

## 📜 Logging

TeamServer logs using `spdlog` in `logs/` and provides good traceability of operations and actions performed. 
For auditability and forensic tracing, TeamServer records the hash for any file payload attached to a task. The log entry includes the filename, payload size, the computed hash, and a timestamp. Hashing is performed in-memory on the payload, and only the fingerprint is persisted in logs.

```log
[2401-06-06 10:14:37.712] [TeamServer] [info] Queued command for beacon MTGAcmSd → 'loadModule upload'
[2401-06-06 10:14:37.712] [TeamServer] [info] File attached to task: 'libUpload.so' | size=34760 bytes | MD5=19cd5908984d71051a17d278a65ee2ec
...
[2401-06-06 10:14:43.429] [TeamServer] [info] Queued command for beacon MTGAcmSd → 'upload superMalware /tmp/superMalware'
[2401-06-06 10:14:43.429] [TeamServer] [info] File attached to task: 'superMalware' | size=3218 bytes | MD5=d98971f190d2f0c2f85bccefab930d33
[2401-06-06 10:14:43.429] [Listener_https_8443_lWdwtgk8] [debug] Queued task for beacon MTGAcmSdMEOg307cOfLdO692WzBORNZG

```

---

## 🛡️ Security Checklist for TeamServer Deployment

Usually, the **TeamServer is deployed inside your own controlled infrastructure**, since it stores and generates **sensitive content** such as operator tasking data. For security and operational control, the TeamServer itself should **not be directly exposed to the internet**.

A common deployment pattern is to place the TeamServer **behind a hardened cloud front** or **reverse proxy**, which acts as the only public-facing component. For example, HTTP(S) or TCP ports exposed by the TeamServer can be **forwarded through a dedicated relay machine** (or VPS) running **Apache2 / Nginx / HAProxy** with strict filtering and logging rules.

This setup provides multiple security benefits:

* 🧱 **Isolation:** Keeps the C2 core off the public internet.
* 🕵️ **Traffic filtering:** The proxy can enforce IP whitelisting, geo-fencing, domain filtering, and rate limiting.
* 🔐 **TLS termination or re-encryption:** Certificates can be managed at the proxy layer, reducing exposure on the TeamServer.
* 🧰 **Stealth:** Makes it easier to blend C2 traffic into normal web traffic using domain fronting or proxy camouflage.
* 📝 **Auditing:** Proxies provide an additional layer of logging and monitoring without touching the C2 codebase.

This layered architecture: **TeamServer in a secured internal environment**, **proxy at the edge**, greatly improves the operational security of your C2 infrastructure while maintaining flexibility for listeners and implants.

---

## What’s next

In Part 2 we’ll deep-dive into **Listeners**:

