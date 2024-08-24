### General case: communications between qubes hosted in two different Qubes OS hosts 

Overview of summarized classes being used and defined in the following sections:

```mermaid
classDiagram
    class BaseVM {
        +name: str
        +app: App
    }

    class LocalVM {
        +name: str
        +app: App
        +start()
        +stop()
        +restart()
    }

    class RemoteVM {
        +name: str
        +app: App
        +connected_relay_vm
    }

    class LocalRelayVM {
        +name: str
        +app: App
        +start()
        +stop()
        +restart()
        +forward_rpc(rpc_request, target_qube)
        +is_relay()
    }

    class RemoteRelayVM {
        +name: str
        +app: App
        +exposed_qubes
        +forward_rpc(target_qube, rpc_request)
        +is_relay()
    }

    BaseVM <|-- LocalVM
    BaseVM <|-- RemoteVM
    LocalVM <|-- LocalRelayVM
    RemoteVM <|-- RemoteRelayVM
```

#### 1. Initial RPC request from Local-Qube to Remote-Qube on Local-QubesOS

This is the starting point where `Local-Qube` initiates an RPC request to `Remote-Qube`, which is a RemoteVM.
`Local-Qube` does not have the knowledge of `Remote-Qube` being a `RemoteVM`.
The request is straightforward and includes the service to be invoked, the argument, and the source and target qube as usual:

```json
{
  "service": "my_service",
  "arg": "my_arg",
  "source": "Local-Qube",
  "target": "Remote-Qube"
}
```

#### 2. RPC Policy process on Local-QubesOS

Local-QubesOS processes the RPC request using its policy engine.
It identifies that `Remote-Qube` is a RemoteVM, meaning it is not on Local-QubesOS but rather accessible through a series of relays `RelayN`, `RelayN-1`, ..., `Relay2` being `RemoteRelayVM` and `Relay1`, a LocalRelayVM on Local-QubesOS.
The communication between `Relay1` and `RelayN` is facilitated using a `transport_method` RPC, lets called it TRANSPORT_RPC, which is an available RPC service on `Relay1`.

#### 3. Making a Qrexec call from Local-Qube to Relay1

To initiate the RPC request, a call is made from `Local-Qube` to `Relay1`.
This request includes all the necessary information to reach `Remote-Qube` via `Relay2`.

```json
{
  "service": "TRANSPORT_RPC",
  "arg": {
    "service": "my_service",
    "arg": "my_arg",
    "source": "Local-Qube",
    "target": "Remote-Qube"
  },
  "source": "Local-Qube",
  "target": "Relay1",
  "destination_relays": [
    "Relay2",
    "Relay3",
    "...",
    "RelayN"
  ]
}
```

#### 4. Execution of TRANSPORT_RPC on Relay1

Once `Relay1` receives the RPC request, it invokes the TRANSPORT_RPC service, passing along the destination relays `Relay2`, `Relay3`, ..., `RelayN` and the original RPC request.

As an illustration, the internal call made by qrexec on `Relay1` is similar to make the following command:
```bash
$ bash /etc/qubes-rpc/<transport_method> <destination_relays> <original_request>
```

> Remark: This would be facilitated by a `forward_rpc` method of the LocalRelayVM.

In this execution, `Relay1` acts as an intermediary, packaging the request and forwarding it to other relays in order.
It is important to note that at this level, forwarding the original request initiated with qrexec to other relays will be delegated to TRANSPORT_RPC service.
This service is able to manage connections with all the relays.
For example, in the case of `ssh`, it is done with [`Proxies and Jump Hosts`](https://en.wikibooks.org/wiki/OpenSSH/Cookbook/Proxies_and_Jump_Hosts).
Local-QubesOS `dom0` will know information about remote relays but is not managing itself connections between relays.
It is up to configurations made inside relays themselves.
In the case of `ssh`, it will be required preliminary setup to copy and share public keys between each jump relay.

#### 5. RelayN forwards the RPC request to Remote-Qube on RemoteHost

TRANSPORT_RPC extracts the original service and arguments and then forwards the request to `Remote-Qube` by doing a qrexec from `RelayN`:

```json
{
  "service": "my_service",
  "arg": "my_arg",
  "source": "RelayN",
  "target": "Remote-Qube",
  "source_relays": [
    "Relay1",
    "Relay2",
    "...",
    "RelayN-1"
  ]
}
```

#### 6. RPC policy process on RemoteHost

On RemoteHost, the RPC policy engine processes the request to ensure that it complies with the allowed policies:

- RemoteHost verifies that `Local-Qube`, via `Relay1`, `Relay2`, ..., `RelayN` is authorized to execute `my_service` with arg `my_arg` on `Remote-Qube`.
- If the policy allows, the request is executed, and the response is sent back along the same relay chain, from `RelayN` to `Relay1`, and finally to `Local-Qube`.

> Remark: The source info needs to be added to https://github.com/QubesOS/qubes-core-qrexec/blob/main/libqrexec/qrexec.h#L139-L143, probably as a new version of this message, for compatibility info.

It's crucial to verify that the correct relay is being used but it as some limitation.
For example, if you have `AppVM1`, `AppVM2`, and `AppVM3` behind `Relay1`, the relay could misrepresent the origin of a request, claiming it came from `AppVM1` when it actually came from `AppVM2`.
However, `Relay1` cannot impersonate a request from `AppVM4`, since `AppVM4` is not configured to route through `Relay1`.
To mitigate such risks, end-to-end verification could be added to ensure the integrity of the request throughout the entire communication chain.
However, achieving full coverage would require unpacking and verifying the request at the target `dom0`, which introduces additional attack surface and complexity to the system.
Such approach is for now let for a further improvement of the mechanisms.

> Remark: Refusing unknown connections is the simplest and most straightforward approach.
> However, in the future, we could consider implementing a mapping system for such connections from specific relays.
> For example, if you have `AppVM1` and `AppVM2` connected via `RelayVM1`, any other unrecognized connection through `RelayVM1` could be mapped to a designated RemoteVM, like `RelayVM1-unknown`.
> This way, unexpected or unconfigured connections are still identifiable and managed, without outright rejection.

#### Summary

```mermaid
sequenceDiagram
    participant LocalQube as Local-Qube
    participant LocalQubesOS as Local-QubesOS
    participant Relay1
    participant Relay2
    participant relayinter as ...
    participant RelayNM1 as RelayN-1
    participant RelayN
    participant RemoteQubesOS as Remote-QubesOS
    participant RemoteQube as Remote-Qube

    LocalQube-->>RemoteQube: 1. Initial RPC request
    LocalQubesOS->>LocalQubesOS: 2. Check RPC Policy
    Note over LocalQubesOS: Remote-Qube is a RemoteVM<br>relayed by RelayN, RelayN-1, ..., Relay1
    LocalQube->>Relay1: 3. Forwarding to a qrexec call from Local-Qube to Relay1
    Relay1-->>RelayN: 4. Execution of TRANSPORT_RPC on Relay1
    RelayN->>RemoteQube: 5. Forward RPC Request to Remote-Qube via Relay2
    RemoteQubesOS->>RemoteQubesOS: 6. Check RPC Policy
    Note over RemoteQubesOS: Local-Qube is a RemoteVM<br>relayed from Relay1
    RemoteQube-->>LocalQube: 7. Deliver Response to Local-Qube
```

### Particular case: registering a RemoteVM not being on a QubesOS host

The general design allows to define and execute RemoteVM being not a "qube" on a Qubes OS host.
Indeed, making `qrexec-client-vm` installable on standard Linux or Windows machines would allow to execute RPC on them.
Such case is particular interesting if you need to interact with for example, KVM virtual machines, RaspberryPi, standard computer with huge resources, computer with different architectures than ones currently supported by Qubes OS, and so on.
It requires some adaptation on `qrexec` codebase to manage policies directly where `qrexec-client-vm` is running instead of `dom0`.

From the general case, assume that we have a LocalRelayVM being `Relay1`, one or more RemoteRelayVM `Relay2`, ..., `RelayN`.
This time, the final `RelayN` is a running compatible `qrexec-client-vm` machine.
In that case, as a RelayVM is a RemoteVM, simply define `RelayN` being a standard RemoteVM with `Remote-Qube` name, and you obtain the following:

```mermaid
sequenceDiagram
    participant LocalQube as Local-Qube
    participant LocalQubesOS as Local-QubesOS
    participant Relay1
    participant RemoteQube as Remote-Qube

    LocalQube-->>RemoteQube: 1. Initial RPC request
    LocalQubesOS->>LocalQubesOS: 2. Check RPC Policy
    Note over LocalQubesOS: Remote-Qube is a RemoteVM<br>relayed by Relay1
    LocalQube->>Relay1: 3. Forwarding to a qrexec call from Local-Qube to Relay1
    Relay1->>RemoteQube: 4. Forward RPC Request to Remote-Qube from Relay1
    RemoteQube->>RemoteQube: 5. Check RPC Policy
    Note over RemoteQube: Local-Qube is a RemoteVM<br>relayed from Relay1
    RemoteQube-->>LocalQube: 6. Deliver Response to Local-Qube
```
