# GoFlow2 Architecture: sFlow Packet Processing

This document explains the journey of an sFlow packet from the moment it arrives at GoFlow2 until it's decoded, processed, and output as JSON or protobuf. 
It also covers how mapping configuration files control what information is extracted and formatted.

## Table of Contents
1. [High-Level Overview](#high-level-overview)
2. [Detailed Processing Pipeline](#detailed-processing-pipeline)
3. [Mapping Configuration](#mapping-configuration)
4. [Data Flow Diagram](#data-flow-diagram)

---

## High-Level Overview

GoFlow2 processes sFlow packets through a multi-stage pipeline:

```
UDP Socket → Receiver → Decoder → Producer → Formatter → Transport
```

Each stage has a specific responsibility:
- **Receiver**: Receives UDP packets and manages worker pools
- **Decoder**: Parses raw bytes into structured sFlow data
- **Producer**: Converts sFlow structures into protobuf messages
- **Formatter**: Serializes messages to JSON/binary/text
- **Transport**: Sends formatted data to stdout/file/Kafka

---

## Detailed Processing Pipeline

### Stage 1: UDP Reception (`utils/udp.go`)

**Entry Point**: `UDPReceiver.Start()`

1. **Socket Creation**: Creates multiple UDP sockets (configurable with `-listen` parameter)
   - Default: `:6343` for sFlow
   - Can create multiple sockets for parallel processing (e.g., `sflow://:6343?count=4`)

2. **Packet Reception**: Each socket runs `receiveRoutine()` in a goroutine
   ```go
   pkt.size, pkt.src, err = udpconn.ReadFromUDP(pkt.payload)
   ```
   - Reads up to 9000 bytes per packet
   - Records reception timestamp
   - Source and destination addresses are captured

3. **Dispatch to Workers**: Packets are sent to a channel (`r.dispatch`)
   - Non-blocking mode: drops packets if channel is full
   - Blocking mode: waits for available workers
   - Uses a memory pool to reuse packet buffers

**Output**: Raw UDP packet wrapped in `Message` struct containing:
- Source address and port
- Destination address and port
- Raw payload bytes
- Reception timestamp

---

### Stage 2: sFlow Decoding (`decoders/sflow/sflow.go`)

**Entry Point**: `DecodeMessageVersion()` → `DecodeMessage()`

#### 2.1 Datagram Header Parsing

Extracts sFlow v5 datagram header:
```go
type Packet struct {
    Version        uint32    // Protocol version (5)
    IPVersion      uint32    // Agent IP version (1=IPv4, 2=IPv6)
    AgentIP        []byte    // Sampling agent address
    SubAgentId     uint32    // Sub-agent ID
    SequenceNumber uint32    // Datagram sequence number
    Uptime         uint32    // Agent uptime in milliseconds
    SamplesCount   uint32    // Number of samples in datagram
    Samples        []interface{}  // Flow or counter samples
}
```

#### 2.2 Sample Iteration

For each sample in the datagram:
1. **Sample Header**: Reads sample format and length
   - Format 1: Flow Sample
   - Format 3: Expanded Flow Sample
   - Format 2/4: Counter Samples (not processed for flows)

2. **Sample Metadata**:
   ```go
   type FlowSample struct {
       SampleSequenceNumber uint32
       SourceIdType         uint32
       SourceIdValue        uint32
       SamplingRate         uint32
       SamplePool           uint32
       Drops                uint32
       Input                uint32  // Input interface
       Output               uint32  // Output interface
       Records              []FlowRecord
   }
   ```

#### 2.3 Flow Record Parsing

Each flow sample contains multiple records. The decoder reads the record type and dispatches to appropriate parser:

**FLOW_TYPE_RAW (1)**: Sampled packet header
```go
type SampledHeader struct {
    Protocol       uint32  // Header protocol (1=Ethernet)
    FrameLength    uint32  // Original frame length
    Stripped       uint32  // Bytes stripped
    OriginalLength uint32  // Length before stripping
    HeaderData     []byte  // Raw packet header (typically 128 bytes)
}
```

**FLOW_TYPE_IPV4/IPV6 (3/4)**: IP-level information
```go
type SampledIPBase struct {
    Length   uint32
    Protocol uint32  // Layer 4 protocol
    SrcIP    []byte
    DstIP    []byte
    SrcPort  uint32
    DstPort  uint32
    TcpFlags uint32
}
```

**FLOW_TYPE_EXT_ROUTER (1002)**: Routing information
```go
type ExtendedRouter struct {
    NextHopIPVersion uint32
    NextHop          []byte
    SrcMaskLen       uint32  // Source prefix length
    DstMaskLen       uint32  // Destination prefix length
}
```

**FLOW_TYPE_EXT_GATEWAY (1003)**: BGP information
```go
type ExtendedGateway struct {
    NextHop           []byte
    AS                uint32
    SrcAS             uint32
    SrcPeerAS         uint32
    ASPath            []uint32    // AS path
    Communities       []uint32    // BGP communities
    LocalPref         uint32
}
```

**FLOW_TYPE_EXT_SWITCH (1001)**: VLAN information
```go
type ExtendedSwitch struct {
    SrcVlan     uint32
    SrcPriority uint32
    DstVlan     uint32
    DstPriority uint32
}
```

**Output**: Structured `Packet` with fully parsed samples and records

---

### Stage 3: Protobuf Production (`producer/proto/producer_sf.go`)

**Entry Point**: `ProcessMessageSFlowConfig()`

#### 3.1 Sample Extraction
```go
GetSFlowFlowSamples(packet)  // Filters flow samples only
```

#### 3.2 Flow Message Creation

For each flow sample, creates a `ProtoProducerMessage` (extends `FlowMessage`):

```go
type ProtoProducerMessage struct {
    flowmessage.FlowMessage  // Protobuf structure from pb/flow.proto
    formatter FormatterMapper
    skipDelimiter bool
}
```

#### 3.3 Record Processing (`SearchSFlowSampleConfig`)

Iterates through each record and populates fields:

**From Sample Header**:
- `Type` = `SFLOW_5`
- `SamplingRate` = sample.SamplingRate
- `InIf` = sample.Input
- `OutIf` = sample.Output
- `Packets` = 1 (always for sFlow sampled headers)

**From SampledHeader Record** → `ParseSampledHeaderConfig()`:
1. **Stores raw header**: `flowMessage.HeaderData = data`
2. **Parses packet layers**: Calls `ParsePacket()` which:
   - Identifies Ethernet frame
   - Extracts MAC addresses → `SrcMac`, `DstMac`
   - Detects EtherType (IPv4/IPv6/802.1q/MPLS)
   - Recursively parses layers

**Layer Parsing Examples**:

- **Ethernet** (`ParseEthernet`):
  ```go
  SrcMac  = binary.BigEndian.Uint64(append([]byte{0,0}, data[6:12]...))
  DstMac  = binary.BigEndian.Uint64(append([]byte{0,0}, data[0:6]...))
  Etype   = binary.BigEndian.Uint16(data[12:14])
  ```

- **802.1q VLAN** (`Parse8021Q`):
  ```go
  VlanId = binary.BigEndian.Uint16(data[0:2]) & 0x0FFF
  ```

- **MPLS** (`ParseMPLS`):
  ```go
  for each label stack entry:
      label = binary.BigEndian.Uint32(data[0:3]) >> 4
      exp   = (data[2] >> 1) & 0x7  // 3 EXP bits
      bos   = data[2] & 1            // Bottom of Stack
      ttl   = data[3]
      
      MplsLabel = append(MplsLabel, label)
      MplsExp   = append(MplsExp, exp)
      MplsBos   = append(MplsBos, bos)
      MplsTtl   = append(MplsTtl, ttl)
  ```

- **IPv4/IPv6** (`ParseIPv4`/`ParseIPv6`):
  ```go
  SrcAddr = data[12:16]  // or [8:24] for IPv6
  DstAddr = data[16:20]  // or [24:40] for IPv6
  Proto   = data[9]      // Layer 4 protocol
  IpTos   = data[1]      // Type of Service
  IpTtl   = data[8]      // Time to Live
  ```

- **TCP/UDP** (`ParseTCP`/`ParseUDP`):
  ```go
  SrcPort  = binary.BigEndian.Uint16(data[0:2])
  DstPort  = binary.BigEndian.Uint16(data[2:4])
  TcpFlags = data[13]  // Only for TCP
  ```

**From ExtendedRouter Record**:
- `NextHop` = nextHop IP
- `SrcNet` = source mask length
- `DstNet` = destination mask length

**From ExtendedGateway Record**:
- `BgpNextHop` = BGP next hop
- `SrcAs` = source AS
- `DstAs` = destination AS (last in AS path)
- `AsPath` = full AS path array
- `BgpCommunities` = BGP community list

**From ExtendedSwitch Record**:
- `SrcVlan` = source VLAN
- `DstVlan` = destination VLAN

**Metadata Added**:
- `SamplerAddress` = packet.AgentIP
- `SequenceNum` = packet.SequenceNumber
- `TimeReceivedNs` = current timestamp in nanoseconds
- `TimeFlowStartNs` = TimeReceivedNs (for sFlow)
- `TimeFlowEndNs` = TimeReceivedNs (for sFlow)

**Layer Stack Recording**:
As each layer is parsed, it's recorded:
```go
LayerStack = ["Ethernet", "MPLS", "IPv6", "TCP"]
LayerSize  = [14, 12, 40, 20]  // Bytes consumed by each layer
```

**Output**: Complete `ProtoProducerMessage` with all decoded fields from protobuf schema

---

### Stage 4: Formatting (`format/json/json.go` or `format/binary/binary.go`)

**Entry Point**: `Format(data interface{})`

#### 4.1 Mapping Configuration Application

**This is where the mapping file controls output!**

The `ProtoProducerMessage` contains a `formatter` field that was configured from the mapping YAML:

```go
type FormatterMapper interface {
    Fields() []string           // List of fields to include
    Render(name string) RenderFunc  // How to format each field
    IsArray(name string) bool   // Is field an array
}
```

**Field Selection** (`formatter.fields` in YAML):
- Only fields listed in `formatter.fields` are included in output
- Omitted fields are completely excluded from JSON/text output
- Binary protobuf includes all fields regardless (wire format)

**Field Rendering** (`formatter.render` in YAML):
The formatter applies renderers to transform raw data:

```go
// Available renderers (producer/proto/render.go)
RendererIP:           IPRenderer           // []byte → "192.168.1.1"
RendererMac:          MacRenderer          // uint64 → "aa:bb:cc:dd:ee:ff"
RendererEtype:        EtypeRenderer        // uint32 → "IPv4"
RendererProto:        ProtoRenderer        // uint32 → "TCP"
RendererDateTime:     DateTimeRenderer     // uint64 → "2025-11-22T..."
RendererDateTimeNano: DateTimeNanoRenderer // uint64 → RFC3339Nano
RendererBase64:       Base64Renderer       // []byte → "YWJjZGVm..."
RendererString:       StringRenderer       // []byte → "string"
```

**Example Transformations**:
```yaml
formatter:
  render:
    src_addr: ip          # [192,168,1,1] → "192.168.1.1"
    src_mac: mac          # 187723572864515 → "aa:bb:cc:dd:ee:ff"
    etype: etype          # 2048 → "IPv4"
    proto: proto          # 6 → "TCP"
    header_data: base64   # [104,229,...] → "aOWeRz0E..."
```

#### 4.2 JSON Marshaling

For JSON format, the `FormatMessageReflectJSON()` method:
1. Iterates through `formatter.Fields()`
2. Retrieves each field value via reflection
3. Applies the configured renderer
4. Builds JSON string manually for efficiency
5. Arrays are formatted as JSON arrays `[item1, item2, ...]`
6. Strings are quoted, numbers are unquoted

**Output**: JSON string like:
```json
{
  "type":"SFLOW_5",
  "time_received_ns":1763848240427771809,
  "sampling_rate":1000,
  "sampler_address":"2001:10:8:50::23",
  "bytes":1512,
  "src_addr":"caba:26::4",
  "dst_addr":"caba:20::4",
  "etype":"IPv6",
  "proto":"TCP",
  "src_port":30044,
  "dst_port":40044,
  "mpls_label":[63400,61400,2],
  "mpls_exp":[0,0,7],
  "mpls_bos":[0,0,1],
  "header_data":"aOWeRz0EaOWeRwcDiEcPeoA/Dv2APw==",
  "layer_stack":["Ethernet","MPLS","IPv6","TCP"]
}
```

---

### Stage 5: Transport (`transport/file/transport.go` or `transport/kafka/kafka.go`)

**Entry Point**: `Send(key, data []byte)`

#### File Transport (default)
- Writes JSON/text to stdout or specified file
- Each message on a new line
- Configurable separator

#### Kafka Transport
- Sends to configured broker and topic
- Uses key for partitioning (if configured)
- Supports compression (gzip, snappy, etc.)

**Output**: Data delivered to final destination

---

## Mapping Configuration

### How Mapping Controls Data Flow

The mapping configuration file (`-mapping=file.yaml`) controls two critical aspects:

#### 1. Field Selection (Reducing Information)

**Purpose**: Control which fields appear in the output to reduce data size and processing overhead.

**Location in Pipeline**: Applied during **Stage 4 (Formatting)**

**Example - Minimal Output**:
```yaml
formatter:
  fields:
    - type
    - time_received_ns
    - src_addr
    - dst_addr
    - src_port
    - dst_port
    - mpls_label
```

**Result**: Only these 7 fields appear in JSON, all others are excluded even if decoded.

**Benefits**:
- Smaller JSON payloads (less bandwidth, storage)
- Faster JSON parsing downstream
- Focus on relevant metrics only

#### 2. Custom Field Extraction (Increasing Information)

**Purpose**: Extract additional data from raw packet bytes that GoFlow2 doesn't parse by default.

**Location in Pipeline**: Applied during **Stage 3 (Production)** packet parsing

**Example - Extract Inner IP Addresses from Encapsulated Packets**:
```yaml
formatter:
  fields:
    - src_ip_encap
    - dst_ip_encap
  protobuf:
    - name: src_ip_encap
      index: 1006
      type: string
      array: true
    - name: dst_ip_encap
      index: 1007
      type: string
      array: true
  render:
    src_ip_encap: ip
    dst_ip_encap: ip
sflow:
  mapping:
    - layer: "ipv4"
      encap: true      # Parse encapsulated IP, not outer IP
      offset: 96       # Bit offset to source IP (12 bytes)
      length: 32       # 32 bits = 4 bytes
      destination: src_ip_encap
    - layer: "ipv4"
      encap: true
      offset: 128      # Bit offset to dest IP (16 bytes)
      length: 32
      destination: dst_ip_encap
```

**What This Does**:
1. During packet parsing (Stage 3), when an encapsulated IPv4 layer is detected
2. Extract bits 96-127 and store in custom field `src_ip_encap`
3. Extract bits 128-159 and store in custom field `dst_ip_encap`
4. During formatting (Stage 4), render these as IP addresses

**Example - Extract Custom NetFlow/IPFIX Fields**:
```yaml
formatter:
  fields:
    - flow_direction
  protobuf:
    - name: flow_direction
      index: 42
      type: varint
ipfix:
  mapping:
    - field: 61          # IPFIX field ID
      destination: flow_direction
```

**What This Does**:
1. During IPFIX decoding (Stage 2), map field ID 61 to custom protobuf field
2. Store in field index 42 (unused in standard schema)
3. Include in formatter output

#### 3. Field Rendering (Changing Representation)

**Purpose**: Control how binary data is displayed in text formats.

**Location in Pipeline**: Applied during **Stage 4 (Formatting)**

**Example**:
```yaml
formatter:
  render:
    time_received_ns: datetimenano  # 1763848240427771809 → "2025-11-22T..."
    src_addr: ip                     # [202,186,0,38,...] → "caba:26::4"
    etype: etype                     # 34525 → "IPv6"
    proto: proto                     # 6 → "TCP"
```

**Without Renderers** (raw protobuf values):
```json
{
  "time_received_ns": 1763848240427771809,
  "src_addr": [202,186,0,38,0,0,0,0,0,0,0,0,0,0,0,4],
  "etype": 34525,
  "proto": 6
}
```

**With Renderers**:
```json
{
  "time_received_ns": "2025-11-22T18:37:20.427771809Z",
  "src_addr": "caba:26::4",
  "etype": "IPv6",
  "proto": "TCP"
}
```

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         UDP PACKET ARRIVES                      │
│                      (sFlow on port 6343)                       │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 1: RECEPTION (utils/udp.go)                              │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ • Multiple UDP sockets (configurable)                  │     │
│  │ • Worker pool receives packets                         │     │
│  │ • Captures: source, dest, timestamp, payload           │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────┬──────────────────────────────────┘
                               │ Message{Src, Dst, Payload, Time}
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 2: DECODING (decoders/sflow/sflow.go)                    │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ DecodeMessageVersion() → DecodeMessage()               │     │
│  │                                                        │     │
│  │ 2.1: Parse Datagram Header                             │     │
│  │      • Version, AgentIP, Sequence, Uptime              │     │
│  │                                                        │     │
│  │ 2.2: For Each Sample                                   │     │
│  │      • Sample format, sequence, sampling rate          │     │
│  │      • Input/output interfaces                         │     │
│  │                                                        │     │
│  │ 2.3: For Each Record in Sample                         │     │
│  │      DecodeFlowRecord()                                │     │
│  │      ├─ FLOW_TYPE_RAW (1)                              │     │
│  │      │  └─ SampledHeader{HeaderData}                   │     │
│  │      ├─ FLOW_TYPE_IPV4/IPV6 (3/4)                      │     │
│  │      │  └─ SampledIP{SrcIP, DstIP, Ports}              │     │
│  │      ├─ FLOW_TYPE_EXT_ROUTER (1002)                    │     │
│  │      │  └─ ExtendedRouter{NextHop, Masks}              │     │
│  │      ├─ FLOW_TYPE_EXT_GATEWAY (1003)                   │     │
│  │      │  └─ ExtendedGateway{AS, BGP data}               │     │
│  │      └─ FLOW_TYPE_EXT_SWITCH (1001)                    │     │
│  │         └─ ExtendedSwitch{VLANs}                       │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────┬──────────────────────────────────┘
                               │ Packet{Samples[]{Records[]}}
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 3: PROTOBUF PRODUCTION (producer/proto/)                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ ProcessMessageSFlowConfig()                            │     │
│  │                                                        │     │
│  │ 3.1: Extract Flow Samples                              │     │
│  │      GetSFlowFlowSamples()                             │     │
│  │                                                        │     │
│  │ 3.2: For Each Sample → Create FlowMessage              │     │
│  │      SearchSFlowSampleConfig()                         │     │
│  │                                                        │     │
│  │ 3.3: Process Each Record                               │     │
│  │      ┌───────────────────────────────────────┐         │     │
│  │      │ SampledHeader Record                  │         │     │
│  │      │ ParseSampledHeaderConfig()            │         │     │
│  │      │                                       │         │     │
│  │      │ • Store HeaderData (raw bytes)        │         │     │
│  │      │ • ParsePacket() - Layer by layer:     │         │     │
│  │      │   ┌────────────────────────────────┐  │         │     │
│  │      │   │ Ethernet → MAC addresses       │  │         │     │
│  │      │   │ 802.1q   → VLAN ID             │  │         │     │
│  │      │   │ MPLS     → Labels, EXP, BoS    │◄─┼─────────┼───  │
│  │      │   │ IPv4/v6  → IPs, ToS, TTL       │  │ MAPPING │     │
│  │      │   │ TCP/UDP  → Ports, Flags        │  │ FILE    │     │
│  │      │   └────────────────────────────────┘  │ CAN ADD │     │
│  │      │                                       │ CUSTOM  │     │
│  │      │ • LayerStack = [...layers]            │ FIELDS  │     │
│  │      │ • LayerSize = [...sizes]              │ HERE    │     │
│  │      └───────────────────────────────────────┘         │     │
│  │      ┌───────────────────────────────────────┐         │     │
│  │      │ ExtendedRouter Record                 │         │     │
│  │      │ • NextHop, SrcNet, DstNet             │         │     │
│  │      └───────────────────────────────────────┘         │     │
│  │      ┌───────────────────────────────────────┐         │     │
│  │      │ ExtendedGateway Record                │         │     │
│  │      │ • ASPath, Communities, BGP data       │         │     │
│  │      └───────────────────────────────────────┘         │     │
│  │      ┌───────────────────────────────────────┐         │     │
│  │      │ ExtendedSwitch Record                 │         │     │
│  │      │ • SrcVlan, DstVlan                    │         │     │
│  │      └───────────────────────────────────────┘         │     │
│  │                                                         │     │
│  │ 3.4: Add Metadata                                      │     │
│  │      • SamplerAddress, SequenceNum, Timestamps         │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────┬──────────────────────────────────┘
                               │ ProtoProducerMessage
                               │ (Complete protobuf with all fields)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 4: FORMATTING (format/json/ or format/binary/)           │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Format(data)                                           │     │
│  │                                                        │     │
│  │ 4.1: Apply Mapping Configuration                       │     │
│  │      ┌────────────────────────────────────────┐        │     │
│  │      │ MAPPING FILE CONTROLS OUTPUT HERE      │◄───────┼──── │
│  │      │                                        │ REDUCE │     │
│  │      │ • Fields to include (formatter.fields) │ INFO   │     │
│  │      │ • Rendering rules (formatter.render)   │        │     │
│  │      └────────────────────────────────────────┘        │     │
│  │                                                        │     │
│  │ 4.2: For Each Field in formatter.fields                │     │
│  │      • Get field value via reflection                  │     │
│  │      • Apply renderer (IP, MAC, base64, etc.)          │     │
│  │      • Convert to JSON/text representation             │     │
│  │                                                        │     │
│  │ Example Renderers:                                     │     │
│  │   src_addr: ip        → [bytes] to "192.168.1.1"       │     │
│  │   src_mac: mac        → uint64 to "aa:bb:cc:dd:ee:ff"  │     │
│  │   etype: etype        → 2048 to "IPv4"                 │     │
│  │   proto: proto        → 6 to "TCP"                     │     │
│  │   header_data: base64 → [bytes] to "aGVsbG8="          │     │
│  │                                                        │     │
│  │ Fields NOT in formatter.fields are EXCLUDED            │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────┬──────────────────────────────────┘
                               │ []byte (JSON, protobuf, or text)
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGE 5: TRANSPORT (transport/file/ or transport/kafka/)       │
│  ┌────────────────────────────────────────────────────────┐     │
│  │ Send(key, data)                                        │     │
│  │                                                        │     │
│  │ File Transport:                                        │     │
│  │ • Write to stdout or file                              │     │
│  │ • One message per line                                 │     │
│  │                                                        │     │
│  │ Kafka Transport:                                       │     │
│  │ • Send to broker/topic                                 │     │
│  │ • Partition by key (optional)                          │     │
│  │ • Compress (gzip, snappy, etc.)                        │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────┬──────────────────────────────────┘
                               ▼
                    ┌────────────────────┐
                    │  FINAL OUTPUT      │
                    │  (JSON, protobuf)  │
                    └────────────────────┘
```

---

## Complete Example: MPLS Packet Processing

Let's trace a real MPLS-labeled IPv6 TCP packet:

### Input: Raw UDP Packet
```
Ethernet: dst=68:e5:9e:47:3d:04 src=68:e5:9e:47:07:03 type=0x8847
MPLS: label=63400 exp=0 bos=0 ttl=63
MPLS: label=61400 exp=0 bos=0 ttl=63
MPLS: label=2 exp=7 bos=1 ttl=63
IPv6: src=caba:26::4 dst=caba:20::4 next=6 hlim=63 tos=246
TCP: sport=30044 dport=40044 flags=0x00
```

### Stage 1: Reception
```
Message{
  Src: "2001:10:8:50::23:6343" # the router sampling the message
  Dst: "[::]:6343" # the collector (goflow2) receiving the samples in any IP on the server
  Payload: [0x68, 0xe5, 0x9e, ...]  // Raw bytes
  Received: 2025-11-22T18:37:20.427771809Z
}
```

### Stage 2: sFlow Decoding
```
Packet{
  Version: 5
  AgentIP: [0x20, 0x01, 0x00, 0x10, ...] // 2001:10:8:50::23
  SequenceNumber: 98324
  Samples: [
    FlowSample{
      SamplingRate: 1000
      Input: 42
      Output: 295
      Records: [
        SampledHeader{
          FrameLength: 1512
          HeaderData: [0x68, 0xe5, 0x9e, 0x47, ...]  // 128 bytes
        }
      ]
    }
  ]
}
```

### Stage 3: Protobuf Production (with Mapping)

**Mapping File** (increases information):
```yaml
sflow:
  mapping:
    # Standard parsing extracts MPLS automatically
    # No custom mapping needed for MPLS
    # Already parsed by ParseMPLS()
```

**Result**:
```
ProtoProducerMessage{
  Type: SFLOW_5
  TimeReceivedNs: 1763848240427771809
  SequenceNum: 98324
  SamplingRate: 1000
  SamplerAddress: [0x20, 0x01, ...]  // 2001:10:8:50::23
  
  InIf: 42
  OutIf: 295
  Bytes: 1512
  Packets: 1
  
  // From Ethernet layer
  SrcMac: 0xaabbccddeeff  // 68:e5:9e:47:07:03
  DstMac: 0x112233445566  // 68:e5:9e:47:3d:04
  Etype: 0x8847           // MPLS
  
  // From MPLS layers (3 labels)
  MplsLabel: [63400, 61400, 2]
  MplsExp:   [0, 0, 7]
  MplsBos:   [0, 0, 1]
  MplsTtl:   [63, 63, 63]
  
  // From IPv6 layer
  SrcAddr: [0xca, 0xba, 0x00, 0x26, ...]  // caba:26::4
  DstAddr: [0xca, 0xba, 0x00, 0x20, ...]  // caba:20::4
  Proto: 6  // TCP
  IpTos: 246
  IpTtl: 63
  Ipv6FlowLabel: 0
  
  // From TCP layer
  SrcPort: 30044
  DstPort: 40044
  TcpFlags: 0
  
  // Raw header (new field!)
  HeaderData: [0x68, 0xe5, 0x9e, ...]  // All 128 bytes
  
  // Layer information
  LayerStack: [Ethernet, MPLS, IPv6, TCP]
  LayerSize: [14, 12, 40, 20]
}
```

### Stage 4: Formatting (with Mapping)

**Mapping File** (reduces and transforms information):
```yaml
formatter:
  fields:
    - type
    - time_received_ns
    - sampling_rate
    - sampler_address
    - bytes
    - src_addr
    - dst_addr
    - src_port
    - dst_port
    - in_if
    - out_if
    - mpls_label
    - mpls_exp
    - mpls_bos
    - header_data
    - layer_stack
  render:
    sampler_address: ip
    src_addr: ip
    dst_addr: ip
    header_data: base64
```

**JSON Output**:
```json
{
  "type": "SFLOW_5",
  "time_received_ns": 1763848240427771809,
  "sampling_rate": 1000,
  "sampler_address": "2001:10:8:50::23",
  "bytes": 1512,
  "src_addr": "caba:26::4",
  "dst_addr": "caba:20::4",
  "src_port": 30044,
  "dst_port": 40044,
  "in_if": 42,
  "out_if": 295,
  "mpls_label": [63400, 61400, 2],
  "mpls_exp": [0, 0, 7],
  "mpls_bos": [0, 0, 1],
  "header_data": "aOWeRz0EaOWeRwcDiEcPeoA/Dv2APwAAL...",
  "layer_stack": ["Ethernet", "MPLS", "IPv6", "TCP"]
}
```

**Note**: Fields like `mpls_ttl`, `tcp_flags`, `ip_tos`, `src_mac`, `dst_mac` were decoded but excluded from output because they weren't in `formatter.fields`.

### Stage 5: Transport
```
Output to stdout:
{"type":"SFLOW_5","time_received_ns":1763848240427771809,...}
```

---

## Summary

**Key Takeaways**:

1. **sFlow packets go through 5 stages**: Reception → Decoding → Production → Formatting → Transport

2. **All decoding happens in Stages 2-3**: The complete packet is parsed and all fields are extracted into the protobuf structure

3. **Mapping file operates at two points**:
   - **Stage 3** (Production): Add custom field extraction via `sflow.mapping` section
   - **Stage 4** (Formatting): Control output fields via `formatter.fields` and `formatter.render`

4. **Reducing information**: Use `formatter.fields` to include only needed fields in JSON output

5. **Increasing information**: Use `sflow.mapping` or `ipfix.mapping` to extract custom fields from raw packet bytes

6. **Transforming information**: Use `formatter.render` to change how fields are displayed (e.g., bytes → base64, IPs → strings)

7. **The protobuf structure always contains all decoded data** - the mapping only controls what gets formatted for text/JSON output

This architecture allows GoFlow2 to be both flexible (custom fields) and efficient (minimal output) while maintaining a clean separation between decoding, production, and formatting.
