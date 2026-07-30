# External userfaultfd backend

The external userfaultfd backend lets Cloud Hypervisor obtain memory contents
from a separate service on demand. The service receives a registered
userfaultfd, monitors page faults, and supplies file-backed ranges without
loading the complete image before VM startup.

The backend uses a binary protocol over an `AF_UNIX`, `SOCK_STREAM` socket.
File descriptors are transferred with `SCM_RIGHTS`, and all multi-byte fields
are little-endian.

## Protocol overview

Every protocol message starts with a 16-byte header.

| Offset | Request field | Response field | Size |
|---:|---|---|---:|
| 0 | `command` | `status` | 2 |
| 2 | `command_headers` | `reply_type` and `reply_headers` | 6 |
| 8 | payload `length` | payload `length` | 8 |

The protocol supports both usage models below. They are client conventions,
not server-enforced connection states, and requests such as `Stat` may be sent
at any time.

| Model | Request and response flow |
|---|---|
| Stateful | `Handshake` transfers a userfaultfd, regions, session flags, and effective UFFD modes; the server resolves faults directly or sends asynchronous `FdRanges` responses |
| Stateless | `Stat`, `Fetch`, `FetchLegacy`, and `Probe` receive matching `Stat`, `FdRanges`, or `Legacy` responses |

| Command | Purpose |
|---|---|
| `Handshake` | Transfer the userfaultfd, regions, session flags, and effective UFFD modes |
| `Stat` | Query flattened-device size and block size |
| `Fetch` | Request file-backed ranges |
| `FetchLegacy` | Request inline data or an existing shared backing |
| `Probe` | Query ranges already available locally |
| `AddRegion` | Add or replace a complete region in a managed session |
| `RemoveRegion` | Remove a complete region from a managed session |

| Reply type | Purpose |
|---|---|
| `Legacy` | Inline data, empty success, or generic error |
| `FdRanges` | File-backed device ranges with attached file descriptors |
| `Stat` | Flattened-device metadata |

The Handshake command header contains `version: u16`, `flags: u8`,
`uffd_modes: u8`, and `region_count: u16`. Each region is a 48-byte record:

| Field | Type | Purpose |
|---|---|---|
| `virt_addr` | `u64` | Start of the registered client VMA |
| `size` | `u64` | Region length |
| `offset` | `u64` | Region offset in the flattened device |
| `fault_size` | `u64` | Preferred fault resolution granularity |
| `prot` | `i32` | Effective mapping protection |
| `flags` | `i32` | Effective mapping flags |
| `backing_offset` | `u64` | Region offset in an optional client backing FD |

Handshake flags select managed fault handling, prefault, an optional Legacy
acknowledgement, and optional client backing FDs. UFFD modes independently
describe `MISSING`, `WP`, and `WP_ASYNC`. A successful Handshake requires no
response unless its `ACK_REQUIRED` flag is set. Without an acknowledgement, a
failed Handshake is reported by closing the connection. With `BACKING_FDS`,
the Handshake carries one backing FD per region after the UFFD, in region
order; `backing_offset` selects the start of each region in its backing FD.

An `FdRanges` response contains one 24-byte
`(device_offset, blob_offset, len)` record per attached FD. Its `MORE` flag
means that another response frame belongs to the same request.

## Cloud Hypervisor operation

The protocol provides two ways for a Cloud Hypervisor client to use an
external backend. `Stat` is optional and may be used whenever the client needs
device metadata.

### Stateful

1. Create a host mapping and register it with userfaultfd in missing-page mode.
2. Connect to the external service and send a Handshake with the userfaultfd
   and VMA regions, session flags, and effective UFFD modes.
3. Let the service monitor faults for the lifetime of the connection.
4. In managed mode, let the service resolve faults directly. Otherwise,
   receive asynchronous `FdRanges` responses.
5. With Cloud Hypervisor's customized handler, map the returned file ranges
   into the registered VMA and wake the faulting thread.
6. Close the socket and join the response thread before destroying the device
   mapping.

### Stateless

1. Connect to the external service without transferring a userfaultfd.
2. Use `Fetch` or `FetchLegacy` to obtain a range, or `Probe` to query data
   already available locally.
3. Map the file ranges returned by `FdRanges`, or copy inline data returned by
   `Legacy`, into the target memory range.

An unexpected disconnect or invalid response in either model is treated as a
backend failure.
