# Guide to Handling Read-Only Parts in OSTree-based Systems

OSTree (e.g., the foundational technology for Red Hat Enterprise Linux for Edge) is a technology that enables **Immutable OS**. Since core system components like the `/usr` directory are read-only, applications, user data, and logs must be placed and stored in areas where writing is permitted.

## 1. Understanding the Read-Only Nature and Its Benefits

* **Scope**: Primarily, the `/usr` directory (containing OS binaries, libraries, system files, etc.) is read-only. This protects the core OS from unintended modifications.
* **Benefits**:
    * **Consistency**: All devices operate from the same "golden image," preventing configuration drift.
    * **Reliability**: Enhanced protection against unauthorized changes and malware.
    * **Atomic Updates and Rollbacks**: OS-wide updates are atomic (succeed or fail completely), allowing for reliable rollback to a previous stable state if issues arise.

## 2. Utilizing Writable Areas for Applications and Data

Applications, user data, and logs should be placed and stored in **writable directories**, separate from the OS's read-only sections.

* **`/var` Directory**:
    * **Purpose**: This is the **most crucial writable area**. It includes variable data, log files, temporary files, runtime data, and container storage.
    * **Examples**: Container root file systems (e.g., `/var/lib/containers`), application data, log files (`/var/log`).

* **`/etc` Directory**:
    * **Purpose**: Stores system-wide configuration files. In OSTree, `/etc` is treated as a configuration overlay, meaning user-modified settings persist even after OS image updates.
    * **Note**: While manual changes are possible, it is best practice to automate and version control configuration management using tools like Ansible. Direct manual changes can lead to configuration discrepancies between devices.

* **`/home` Directory**:
    * **Purpose**: User home directories, where user-specific settings and data are stored.
    * **Note**: In edge devices, it's less common for multiple interactive users to log in as they would on a traditional server, but it can be used for user home directories if needed.

* **`/opt` Directory (Lower Recommendation)**:
    * **Purpose**: Traditionally used for installing third-party software.
    * **Note**: In an OSTree environment, **containerization of applications is strongly recommended**. Direct installation to `/opt` should be reserved for rare, exceptional cases where containerization is extremely difficult.

## 3. Emphasizing Containerization (Most Important)

**Containerization** is the most effective solution for running applications in a read-only OS environment.

* **Reason**: Containers possess their own writable layers, completely isolated from the host OS's read-only sections.
* **Mechanism**: Inside a container, an application behaves as if it has its own independent file system. The container's writable layers are stored on the host OS in directories like `/var/lib/containers`.
* **Benefits**:
    * Enables flexible application deployment and updates while maintaining the OS's immutability.
    * Application dependencies are encapsulated within the container, preventing impact on the host OS.
    * Simplifies application lifecycle management (deployment, updates, rollbacks).
* **Implementation**: Utilize Podman (for single devices) or MicroShift/Kubernetes (for multiple containers/devices).

## 4. Designing for Persistent Storage

If applications generate data or require data to persist across reboots or updates, persistent storage must be designed.

* **For Kubernetes (MicroShift)**:
    * **Persistent Volume (PV) / Persistent Volume Claim (PVC)**: Utilize the device's local storage (under `/var`, for example) as a Kubernetes persistent volume and mount it to containers.
    * **Host Path Volume**: Directly map a path inside the container to a writable directory on the host OS, such as `/var`.
* **For Podman**:
    * **Volume Mounts**: Use commands like `podman run -v /host/path:/container/path ...` to mount a host path into the container, ensuring data persistence in a writable area of the host OS.

## 5. Automating Configuration Management

* When host OS configurations in `/etc` need to be modified, use configuration management tools like Ansible to **declaratively apply and manage settings**. This prevents configuration drift from manual changes and ensures consistent settings across all edge devices.

## Summary

The primary strategy for handling the read-only parts in OSTree-based systems is **thorough containerization of applications** and the **appropriate utilization of standard writable OS directories** such as `/var` and `/etc`. This approach ensures both OS robustness and application flexibility.
