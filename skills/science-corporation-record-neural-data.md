---
name: Record neural data from a Synapse device
description: >-
  Discover a Science Corporation Synapse neural interface device on the local network,
  configure a broadband signal chain, start acquisition, and read the recorded data — using
  the synapsectl CLI / Synapse Python client over the SynapseDevice gRPC service.
api: grpc/science-corporation-synapse.proto
method: generated
source: https://science.xyz/docs/d/synapse/quickstart
operations: [discover, Info, Configure, Start, Query, StreamQuery, Stop, ReadFile]
---

# Record neural data from a Synapse device

This skill drives a Synapse device on the local network. Synapse is a device-local gRPC API
(package `synapse`, service `SynapseDevice`); there is no cloud endpoint and no API key —
connect by device name or IP. Data streams over UDP (NDTP), so raise the OS UDP buffer
(`net.core.rmem_max` / `wmem_max`) before high-throughput reads.

Install: `pip install science-synapse` (provides the `synapsectl` CLI and the `synapse-sim`
simulator for local development).

## Steps

1. **Discover** the device on the network: `synapsectl discover`. Note its name / IP for
   `--uri`.
2. **Info** — call `SynapseDevice.Info` (`synapsectl --uri <dev> info`) to read `DeviceInfo`:
   its `serial`, `synapse_version`/`firmware_version`, attached `peripherals`, and current
   `Status`. Confirm `DeviceState` is `kStopped` before configuring.
3. **Configure** — build a `DeviceConfiguration` (a list of typed `NodeConfig` nodes plus
   `NodeConnection` edges) and call `SynapseDevice.Configure`. A minimal recording chain
   wires a `kBroadbandSource` node to a `kDiskWriter` (or exposes a Tap). Check the returned
   `Status.code` is `kOk`; `kInvalidConfiguration` means fix the node/connection graph.
4. **Start** — call `SynapseDevice.Start`. On success `DeviceState` becomes `kRunning`.
5. **Read the data** — either subscribe with `SynapseDevice.StreamQuery` for live
   `BroadbandFrame`s, or use `synapsectl read` to pull a Broadband Tap to an HDF5 file. Use
   each frame's `sequence_number` to detect dropped UDP frames.
6. **Stop** — call `SynapseDevice.Stop` when done; `DeviceState` returns to `kStopped`.
7. **Retrieve files** — `SynapseDevice.ListFiles` then `ReadFile` (streamed) to pull any
   DiskWriter output off the device.

## Error handling

Every mutating RPC returns a `Status` (`code`: `StatusCode`, `message`, `state`). Handle:
`kInvalidConfiguration` (fix the signal chain), `kFailedPrecondition` (wrong `DeviceState` —
e.g. Start before Configure), `kPermissionDenied`, `kQueryFailed`. See
`errors/science-corporation-error-codes.yml` and `conventions/science-corporation-conventions.yml`.
